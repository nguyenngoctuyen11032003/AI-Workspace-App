# CONFLICTS_TO_RESOLVE — Phiên bản 2 (triage Phase 0, Decision Analyst)

Ngày: 2026-10-30. Phiên bản 2 nằm ở đầu file này; nội dung phiên bản 1 (System Architect, 55 dòng) được giữ nguyên bên dưới dấu phân cách. Mã CF-001..CF-055 không đổi.

**Quy tắc bắt buộc:** chủ sở hữu (owner) chưa duyệt bất kỳ quyết định nào. Không dòng nào dưới đây là APPROVED. Cột "Khuyến nghị" chỉ là đề xuất của analyst. Mọi chỉnh sửa tài liệu có điều kiện `SAU D-xxx` chỉ được áp dụng sau khi owner ghi `APPROVED` cho D-xxx trong `PHASE_0_OWNER_DECISION_REGISTER.md`; điều kiện `NGAY` nghĩa là chỉnh sửa đúng với mọi kết quả quyết định (chỉ sửa tên, đường dẫn, mâu thuẫn nội bộ).

Phương pháp xác minh: mỗi mâu thuẫn được grep lại ở file:dòng được trích (2026-10-30). Kiểm chứng thực tế bổ sung trong phiên này: `docker info` (daemon vẫn không chạy: pipe `dockerDesktopLinuxEngine` không tồn tại; Docker CLI 29.8.1), `where codex` (không có), `npm view drizzle-orm dist-tags.latest` = 0.45.4 (1.0 chỉ rc/beta), `npm view phaser` latest = 4.2.1, `claude auth status` có `CLAUDE_CONFIG_DIR` rỗng trả `loggedIn:false`, mặc định `loggedIn:true` (không đọc file credential). itch.io trả HTTP 403 cho WebFetch nên giấy phép JIK-A-4 chưa kiểm được.

## 0. Từ vựng và số liệu

Phân loại (đúng một): `TECH_EVIDENCE_RESOLVABLE` (bằng chứng trong repo/lệnh quyết định được, không cần sở thích owner), `OWNER_DECISION` (cần chọn về rủi ro, chi phí, phạm vi, hoặc đổi yêu cầu gốc của owner), `STALE_DOC_SYNC` (tài liệu lệch so với tài liệu chính tắc hoặc quyết định đã đề xuất; chỉ cần đồng bộ).
Trạng thái (chỉ dùng): `PENDING_OWNER_APPROVAL`, `RESOLVED_BY_EVIDENCE`, `STALE_FIX_PENDING`, `ALREADY_FIXED`.

| Phân loại | Số lượng | Trạng thái |
|---|---|---|
| OWNER_DECISION | 15 | 15 PENDING_OWNER_APPROVAL |
| TECH_EVIDENCE_RESOLVABLE | 7 | 7 RESOLVED_BY_EVIDENCE (vẫn cần đồng bộ tài liệu; chỉnh sửa nêu bên dưới) |
| STALE_DOC_SYNC | 33 | 31 STALE_FIX_PENDING + 2 ALREADY_FIXED (CF-012, CF-033) |
| Tổng | 55 | |

**Vì sao khác tỷ lệ v1 (37 "Có" / 18 "Không")**: v1 gắn "Có" cho mọi mâu thuẫn mà đề xuất của System Architect còn phụ thuộc một D-xxx chưa duyệt. Phiên bản 2 tách hai việc: (1) quyết định chính (D-001..D-020) owner duyệt một lần trong register; mâu thuẫn chỉ cần đồng bộ tài liệu theo D-xxx đó được xếp `STALE_DOC_SYNC` có điều kiện `SAU D-xxx` (không hỏi owner hai lần); (2) chỉ những mâu thuẫn đưa ra câu hỏi mới ngoài D-xxx, hoặc mâu thuẫn với yêu cầu gốc của owner (FR đánh dấu [Owner], "chỉ CC0", "không chạy code AI trên host"), hoặc cần owner chấp nhận rủi ro/chi phí, mới là `OWNER_DECISION` (15). 7 dòng giải được bằng bằng chứng trong repo hoặc lệnh. CF-012 và CF-033 đã được sửa hoặc không còn tồn tại. Ngoài ra v2 sửa ba chỗ so với đề xuất v1: CF-004 (v1 đề xuất con trỏ không có outbox, nhưng EVENT_SYSTEM §7.1 và D-014 dùng bảng `event_outbox`, và `bigserial` không đúng thứ tự commit nên con trỏ có nguy cơ bỏ sót), CF-038 (`maxAttempts=3` tương đương `max_retry=2`) và CF-042 (UUID v7 cần PG 18, ADR-0002 chọn PG 16).

## 1. Bảng tóm tắt

| ID | Chủ đề | Phân loại | Tag | Phụ thuộc | Trạng thái |
|---|---|---|---|---|---|
| CF-001 | ORM Prisma vs Drizzle | STALE_DOC_SYNC | architecture | D-002 | STALE_FIX_PENDING |
| CF-002 | Concurrency mặc định | STALE_DOC_SYNC | architecture, cost | D-010 | STALE_FIX_PENDING |
| CF-003 | Quy ước tên event | OWNER_DECISION | architecture | D-014, FR-117 | PENDING_OWNER_APPROVAL |
| CF-004 | Outbox vs event store | TECH_EVIDENCE_RESOLVABLE | architecture | D-014 | RESOLVED_BY_EVIDENCE |
| CF-005 | Tên bảng audit | STALE_DOC_SYNC | architecture, security | không | STALE_FIX_PENDING |
| CF-006 | Retention audit | OWNER_DECISION | security, cost | D-002 | PENDING_OWNER_APPROVAL |
| CF-007 | Timeout approval | OWNER_DECISION | security, architecture | D-003, D-004 | PENDING_OWNER_APPROVAL |
| CF-008 | Mức quyền L0-L5 vs enum | OWNER_DECISION | security | D-016, D-007 | PENDING_OWNER_APPROVAL |
| CF-009 | agent_status 12/5/8 | STALE_DOC_SYNC | architecture | D-017 | STALE_FIX_PENDING |
| CF-010 | node_run_status | TECH_EVIDENCE_RESOLVABLE | architecture | D-003 | RESOLVED_BY_EVIDENCE |
| CF-011 | Case enum task_status | OWNER_DECISION | architecture | D-017 | PENDING_OWNER_APPROVAL |
| CF-012 | provider_kind thiếu giá trị | STALE_DOC_SYNC | architecture | D-005, D-006 | ALREADY_FIXED |
| CF-013 | Cổng approval Claude | STALE_DOC_SYNC | security | D-004 | STALE_FIX_PENDING |
| CF-014 | Chế độ spawn Claude | STALE_DOC_SYNC | security | D-004 | STALE_FIX_PENDING |
| CF-015 | Approval policy Codex | STALE_DOC_SYNC | security | D-005 | STALE_FIX_PENDING |
| CF-016 | Container warm vs ephemeral | STALE_DOC_SYNC | security, architecture | D-007 | STALE_FIX_PENDING |
| CF-017 | POL-11/12 vs R030/R040/R050 | OWNER_DECISION | security | D-009 | PENDING_OWNER_APPROVAL |
| CF-018 | Engine Office | STALE_DOC_SYNC | architecture | D-011 | STALE_FIX_PENDING |
| CF-019 | Tile size và map | TECH_EVIDENCE_RESOLVABLE | architecture | D-011 | RESOLVED_BY_EVIDENCE |
| CF-020 | Số animation | TECH_EVIDENCE_RESOLVABLE | MVP-scope | D-011 | RESOLVED_BY_EVIDENCE |
| CF-021 | Màu blocked | STALE_DOC_SYNC | architecture | D-015 | STALE_FIX_PENDING |
| CF-022 | Relax/Coffee khi idle | OWNER_DECISION | MVP-scope | D-011 | PENDING_OWNER_APPROVAL |
| CF-023 | waiting_approval đứng đâu | STALE_DOC_SYNC | MVP-scope | D-011 | STALE_FIX_PENDING |
| CF-024 | OpenHands ACP vs D-008 | OWNER_DECISION | security | D-006, D-008 | PENDING_OWNER_APPROVAL |
| CF-025 | Giấy phép sprite JIK-A-4 | STALE_DOC_SYNC | security (pháp lý) | D-012 | STALE_FIX_PENDING |
| CF-026 | CC0-only vs D-012 | OWNER_DECISION | security (pháp lý), MVP-scope | D-012 | PENDING_OWNER_APPROVAL |
| CF-027 | Turborepo | STALE_DOC_SYNC | architecture | D-001 | STALE_FIX_PENDING |
| CF-028 | Test runner | STALE_DOC_SYNC | architecture | D-001 | STALE_FIX_PENDING |
| CF-029 | Số loại node workflow | OWNER_DECISION | MVP-scope | D-003 | PENDING_OWNER_APPROVAL |
| CF-030 | Template mặc định | TECH_EVIDENCE_RESOLVABLE | MVP-scope, security | D-003 | RESOLVED_BY_EVIDENCE |
| CF-031 | PM chấm điểm | OWNER_DECISION | cost, MVP-scope | D-003, D-016 | PENDING_OWNER_APPROVAL |
| CF-032 | Tên branch/worktree | STALE_DOC_SYNC | architecture | D-009 | STALE_FIX_PENDING |
| CF-033 | Vị trí DECISION_LOG | STALE_DOC_SYNC | architecture | không | ALREADY_FIXED |
| CF-034 | Đường dẫn SECURITY/DATABASE_DESIGN | STALE_DOC_SYNC | architecture | không | STALE_FIX_PENDING |
| CF-035 | Thiếu bảng event_outbox | STALE_DOC_SYNC | architecture | D-014 | STALE_FIX_PENDING |
| CF-036 | Thiếu bảng template/notification_reads | STALE_DOC_SYNC | architecture | D-016, D-017 | STALE_FIX_PENDING |
| CF-037 | Thiếu AC cho nhóm FR MVP | STALE_DOC_SYNC | MVP-scope | không | STALE_FIX_PENDING |
| CF-038 | Số lần retry | STALE_DOC_SYNC | cost | D-010 | STALE_FIX_PENDING |
| CF-039 | Fallback giữa provider | OWNER_DECISION | cost, security | D-010, D-004 | PENDING_OWNER_APPROVAL |
| CF-040 | Tên interface adapter | STALE_DOC_SYNC | architecture | không | STALE_FIX_PENDING |
| CF-041 | Xác thực | STALE_DOC_SYNC | security | D-001 | STALE_FIX_PENDING |
| CF-042 | Quy ước đặt tên hạ tầng | STALE_DOC_SYNC | architecture | D-001, D-014 | STALE_FIX_PENDING |
| CF-043 | CLI trên host (D-007) | OWNER_DECISION | security | D-007 | PENDING_OWNER_APPROVAL |
| CF-044 | 18 agent vs quota/ToS | OWNER_DECISION | cost, security | D-010, D-004 | PENDING_OWNER_APPROVAL |
| CF-045 | Docker/Codex chưa kiểm chứng | OWNER_DECISION | MVP-scope, security | D-005, D-007 | PENDING_OWNER_APPROVAL |
| CF-046 | Schema manifest quyền file | STALE_DOC_SYNC | security | D-009 | STALE_FIX_PENDING |
| CF-047 | Cấu hình/env Claude | TECH_EVIDENCE_RESOLVABLE | security | D-004, D-008 | RESOLVED_BY_EVIDENCE |
| CF-048 | WAITING_APPROVAL có chiếm slot | STALE_DOC_SYNC | architecture, cost | D-010 | STALE_FIX_PENDING |
| CF-049 | API đọc quota | STALE_DOC_SYNC | cost | D-010 | STALE_FIX_PENDING |
| CF-050 | Tên event approval Codex | STALE_DOC_SYNC | security | D-005 | STALE_FIX_PENDING |
| CF-051 | Hash chain audit | STALE_DOC_SYNC | security | D-008 | STALE_FIX_PENDING |
| CF-052 | Bảng secrets vs credential_ref | STALE_DOC_SYNC | security | D-008 | STALE_FIX_PENDING |
| CF-053 | Worktree host-truth | TECH_EVIDENCE_RESOLVABLE | architecture, security | D-007, D-009 | RESOLVED_BY_EVIDENCE |
| CF-054 | Tái dùng UI OpenHands | STALE_DOC_SYNC | security (pháp lý) | D-012, D-006 | STALE_FIX_PENDING |
| CF-055 | approverRoles vs RBAC | STALE_DOC_SYNC | security | D-003 | STALE_FIX_PENDING |

## 2. Chi tiết từng mâu thuẫn

Ký hiệu: P = ưu điểm, N = nhược điểm, H = hệ quả. "Sửa:" liệt kê `file:dòng, cũ -> mới, điều kiện`. Số dòng đúng ngày 2026-10-30; nếu dòng lệch, tìm theo chuỗi `cũ`.

### CF-001 ORM: Prisma vs Drizzle
- Tài liệu: `research/ORCHESTRATION_WORKFLOW_RESEARCH.md:4,210-212,351,399,412,561,671,686`; `architecture/AGENT_ORGANIZATION.md:41,42,66,67,138`; `security/PERMISSION_MODEL.md:101,120`; `SECURITY.md:241,376,430,432`; `development/ENVIRONMENT_SETUP.md:178`; `WORK_PLAN.md` (dòng "Prisma/Drizzle"); `research/MY_VIRTUAL_OFFICE_ANALYSIS.md:461` vs `DATABASE_DESIGN.md:29-52,1468` và D-002.
- Mâu thuẫn (đã xác minh bằng grep, vẫn còn): đường dẫn quyền sở hữu `prisma/**` làm diff-gate bảo vệ sai thư mục. Phát hiện thêm: ADR-0002 nói `packages/db/**`, `DEVELOPMENT_GUIDE.md:61,63,107` dùng `packages/db`, nhưng `DATABASE_DESIGN.md:57,1282-1283` đặt schema ở `apps/api/src/database/**`. Nhãn "Drizzle (v1.x)" (DATABASE_DESIGN:29) sai: `npm view` cho thấy stable = 0.45.4, 1.0 mới ở beta/rc.
- Phương án: A) giữ Drizzle, đồng bộ tài liệu (P: khớp D-002, hợp partition/partial index; N: chưa có prototype partition, API quan hệ kém chín; H: đường dẫn `apps/api/src/database/**`, skill `drizzle`). B) Prisma (P: hệ sinh thái; N: không mô tả partition/trigger, dịch lại 44 bảng; H: sửa D-002).
- Khuyến nghị: A, ghim 0.45.x và chạy spike partition 1 ngày trước khi khóa (ADR-0002). Dùng `apps/api/src/database/**` (khớp DATABASE_DESIGN) thay vì `packages/db/**`, rồi sửa ADR-0002 và DEVELOPMENT_GUIDE.
- Sửa (SAU D-002): AGENT_ORGANIZATION:41,42,66,67 `prisma/**` -> `apps/api/src/database/**`; :138 `code: 'prisma'` -> `code: 'drizzle'`; PERMISSION_MODEL:101 `prisma/schema.prisma` -> `apps/api/src/database/schema/**`; :120 `prisma migrate` -> `drizzle-kit migrate`; SECURITY:241 `prisma migrate (deploy|reset)` -> `drizzle-kit (migrate|push|drop)`; :376 `schema.prisma` -> `apps/api/src/database/schema/**`; :430 `prisma/migrations/2026*_billing*/**` -> `apps/api/src/database/migrations/2026*_billing*/**`; :432 `prisma/schema.prisma` -> `apps/api/src/database/schema/**`; ENVIRONMENT_SETUP:178 `(Prisma/Drizzle/TypeORM)` -> `(Drizzle, D-002)`; DATABASE_DESIGN:29 `Drizzle (v1.x)` -> `Drizzle (0.45.x stable; 1.0 beta)`; WORK_PLAN "Prisma/Drizzle" -> "Drizzle"; ORCHESTRATION_WORKFLOW_RESEARCH:4 `PostgreSQL (Prisma)` -> `PostgreSQL (Drizzle; mã Prisma trong tài liệu chỉ minh họa)`; DEVELOPMENT_GUIDE:61,63,107 `packages/db` -> `apps/api/src/database` (hoặc sửa DATABASE_DESIGN:57 nếu owner chọn `packages/db`).

### CF-002 Concurrency mặc định
- Tài liệu: `DATABASE_DESIGN.md:324` (`max_concurrent` default 2), `:1318` (seed `max_concurrent_agents = 3`); `research/CLAUDE_CODE_INTEGRATION.md:724` (`MAX_PARALLEL_AGENTS=10`); `MULTI_AGENT_ORCHESTRATION.md:322` (M-O2 đã hòa giải); `architecture/WORKSPACE_AND_GIT_ISOLATION.md:78,199` (worktree 6/12) vs D-010 (5, tối đa 18).
- Mâu thuẫn còn thật: seed 3 và 10 chưa vá; default provider 2 chỉ là giá trị khởi tạo.
- Phương án: A) đồng bộ mọi nơi theo D-010 (P: một nguồn; N: không). B) giữ số khác nhau theo ngữ cảnh (P: ít sửa; N: người triển khai lấy nhầm).
- Khuyến nghị: A.
- Sửa (SAU D-010): DATABASE_DESIGN:1318 `max_concurrent_agents = 3` -> `max_concurrent_agents = 5`; :324 `.default(2)` -> `.default(2) // khởi tạo; cap thực lấy từ ResourceConfig (claude 5, codex 3)`; CLAUDE_CODE_INTEGRATION:724 `MAX_PARALLEL_AGENTS=10` -> `MAX_PARALLEL_AGENTS=5 (mặc định, trần 18; D-010)`; WORKSPACE_AND_GIT_ISOLATION:78,199: thêm chú thích `giới hạn worktree là cấu hình riêng, không phải trần agent`.

### CF-003 Quy ước tên event
- Tài liệu: `EVENT_SYSTEM.md:39,72,172,407 (M-E2)` chọn UPPER_SNAKE vì FR-117; `SYSTEM_ARCHITECTURE.md:303 (#3)` và ADR-0014 chọn dotted lowercase; `DATABASE_DESIGN.md:1190-1200` dotted; `VIRTUAL_OFFICE.md:156-162,368` dùng cả hai. QA P1-13 khuyến nghị UPPER_SNAKE + quy đổi.
- Mâu thuẫn thật: hai tài liệu chính tắc (EVENT_SYSTEM và SYSTEM_ARCHITECTURE/ADR-0014) chọn ngược nhau; FR-117 là yêu cầu của owner dùng tên UPPER_SNAKE.
- Phương án: A) `type` lưu dotted lowercase, hiển thị UPPER_SNAKE bằng bảng ánh xạ 1-1 trong `@aiw/contracts` (P: khớp DB, audit, Redis channel; N: hai dạng tên, cần test ánh xạ; H: sửa M-E2). B) UPPER_SNAKE ở mọi nơi (P: khớp FR-117 và code mẫu A07; N: migration `agent_events.type` và index; H: sửa DATABASE_DESIGN §5.2, AUDIT_LOGGING). C) giữ hai quy ước không ánh xạ (loại: lệch contract).
- Khuyến nghị: A, vì FR-117 vẫn thỏa ở lớp hiển thị và API.
- Câu hỏi owner: "FR-117 liệt kê tên event UPPER_SNAKE (vd `TASK_BLOCKED`). Bạn chấp nhận lưu nội bộ dạng dotted lowercase (`task.blocked`) và chỉ hiển thị/xuất API dạng UPPER_SNAKE, hay muốn UPPER_SNAKE ở cả DB?" Mặc định đề xuất: A.

### CF-004 Outbox vs event store append-only
- Tài liệu: `EVENT_SYSTEM.md:222-227,247 (§7.1)`, `DATABASE_DESIGN.md:1231,1471 (§5.4)`, ADR-0014, D-014, D-002.
- Bằng chứng: EVENT_SYSTEM:226 nêu role API chỉ có `INSERT, SELECT` nên không đặt cột `published`; :247 nêu `bigserial` không bảo đảm thứ tự commit. Đề xuất v1 (một relay đọc theo con trỏ `global_seq`, không bảng outbox) có thể bỏ sót dòng có `global_seq` nhỏ hơn con trỏ nhưng commit muộn; bảng `event_outbox` mà relay `DELETE` sau khi publish không có lỗ hổng đó, và khớp chữ "outbox" của D-014.
- Phương án: A) bảng `event_outbox(event_id, global_seq)` + một leader relay (advisory lock) at-least-once (P: không mất event, giữ append-only; N: thêm 1 bảng, 1 insert mỗi event; H: 45 bảng). B) con trỏ `global_seq` (P: ít ghi; N: nguy cơ bỏ sót vì thứ tự commit). C) publish-after-commit (P: đơn giản; N: mất event khi crash).
- Khuyến nghị: A, dùng ở mọi chế độ (EVENT_SYSTEM §7.1).
- Sửa (SAU D-014): ADR-0014 "relay đọc event append-only theo `global_seq` (CF-004, CF-035)" -> "relay đọc bảng `event_outbox` (CF-004, CF-035)"; CONFLICTS v1 dòng CF-004/CF-035 coi là bị thay thế bởi v2.

### CF-005 Tên bảng audit
- Tài liệu: `SECURITY.md:484`; `architecture/INFRASTRUCTURE_DESIGN.md:185,212`; `development/DEPLOYMENT.md:125,143,163,182`; `operations/DEVSECOPS_GUIDE.md:116,174,207`; `architecture/SKILL_SYSTEM.md:497,645` (`audit_log`) vs `security/AUDIT_LOGGING.md:44`, DATABASE_DESIGN (`audit_logs`). Còn nguyên (đã grep).
- Phương án: A) sửa mọi nơi thành `audit_logs` (P: một tên; N: không). B) đổi DB thành `audit_events` (N: đổi 44 bảng, trigger, migration).
- Khuyến nghị: A. NGAY.
- Sửa (NGAY): thay `audit_events` -> `audit_logs` tại SECURITY:484, INFRASTRUCTURE_DESIGN:185,212, DEPLOYMENT:125,143,163,182, DEVSECOPS_GUIDE:116,174,207; SKILL_SYSTEM:497,645 `audit_log` -> `audit_logs`. SECURITY:484: thêm câu `Chi tiết schema: security/AUDIT_LOGGING.md §4`.

### CF-006 Retention audit
- Tài liệu: `SECURITY.md:498` (180 ngày) vs `DEVSECOPS_GUIDE.md:116,170,207` và `AUDIT_LOGGING.md:158,239` (365 ngày + export ký 2 năm); AUDIT_LOGGING đánh dấu cần phê duyệt.
- Phương án: A) 365 ngày online + export JSONL ký HMAC thêm 2 năm (P: điều tra dài hạn, đủ cho server mode; N: tốn dung lượng đĩa; H: partition detach). B) 180 ngày (P: nhẹ; N: ngắn cho điều tra sự cố chậm). C) 90 ngày (loại: không đủ cho forensics).
- Khuyến nghị: A (đĩa D: còn ~108 GB; audit 5-30 dòng/giây cao điểm là giả định).
- Câu hỏi owner: "Audit log giữ online bao lâu: 365 ngày (khuyến nghị, kèm export ký) hay 180 ngày?" Sau khi duyệt, sửa SECURITY:498 `180 ngày audit` -> giá trị đã chọn.

### CF-007 Timeout approval
- Tài liệu: `SECURITY.md:400` (540 s), `BACKEND_ARCHITECTURE.md:149`, `DATABASE_DESIGN.md:832` (3600 s), `PROVIDER_INTEGRATION.md:411` (10 phút), `architecture/RESOURCE_MANAGEMENT.md:104` (24 h), `product/USER_ROLES_AND_PERMISSIONS.md` §5.5 (không timeout, nhắc 24 h), `WORKFLOW_ENGINE.md:201,359` (null), research Claude (30 phút). Sáu giá trị, xác minh còn nguyên.
- Phương án: A) theo tầng: hook HTTP/tool-level chờ tối đa 540 s rồi trả `deny pending_human`, task sang `WAITING_HUMAN`, approval vẫn `pending`, kích hoạt lại khi duyệt; workflow-level không timeout, nhắc 24 h; `prod_deploy/data_deletion/merge_main` luôn `scope=once`, ttl <= 15 phút (P: không giữ slot lâu, khớp D-003; N: phải thiết kế resume, kích hoạt lại tốn spawn ~48k token; H: sửa 6 nơi). B) một timeout tool-level 10 phút rồi `BLOCKED` (P: đơn giản; N: task mất ngữ cảnh khi owner vắng). C) không timeout (N: giữ slot vô hạn).
- Khuyến nghị: A.
- Câu hỏi owner: "Khi agent chờ bạn duyệt: (a) chấp nhận chờ tối đa 540 s giữ process, sau đó task chuyển WAITING_HUMAN và chạy lại khi bạn duyệt (tốn thêm một lần spawn); (b) nhắc sau 24 giờ; (c) lệnh deploy production/xóa dữ liệu/merge main luôn cần duyệt từng lần, hết hạn sau 15 phút?"

### CF-008 Mức quyền L0-L5 vs enum
- Tài liệu: `USER_ROLES_AND_PERMISSIONS.md` §4.2 (L0-L5), `AGENT_ARCHITECTURE.md:148,276`, `architecture/AGENT_ORGANIZATION.md:30` (L0..L4), `DATABASE_DESIGN.md:170` (4 giá trị, có `full`), `security/PERMISSION_MODEL.md:36-48,309,321`.
- Phương án: A) L0-L5 là nguồn, enum DB giữ 4 giá trị cộng cột `permission_profile` (P: sửa tối thiểu; N: hai cấp độ cần ánh xạ). B) mở rộng enum thành 6 (P: một nguồn; N: migration, enum khó đổi sau). C) bỏ L5/`full` (P: ít bề mặt tấn công; N: mất mức cho break-glass).
- Khuyến nghị: A, cấm `full` bằng CHECK, chốt số level trước P1-04.
- Câu hỏi owner: "Duyệt thang quyền agent L0-L5 ánh xạ sang 3 giá trị DB (`read_only`, `standard`, `elevated`) + profile, và cấm hẳn giá trị `full`?"

### CF-009 agent_status 12 vs 5 vs 8
- Tài liệu: `SYSTEM_ARCHITECTURE.md:306`, `BACKEND_ARCHITECTURE.md:205`, `design/COMPONENT_SPECIFICATION.md:334`, `design/SCREEN_SPECIFICATION.md:19`; DATABASE_DESIGN:167 (12). Còn thật: thiếu bảng ánh xạ chính thức.
- Phương án: A) 12 là enum; 12 -> 5 attention -> 8 tone là hàm thuần trong `packages/contracts` (ADR-0017). B) đổi enum DB (loại).
- Khuyến nghị: A.
- Sửa (SAU D-017): BACKEND_ARCHITECTURE:205 `cần xác nhận với Database agent.` -> `đã chốt: 12 giá trị là enum; phép chiếu 12->5->8 ở packages/contracts (ADR-0017).`; SYSTEM_ARCHITECTURE:306 thêm cuối ô `(chốt: ADR-0017)`; COMPONENT_SPECIFICATION:334 thêm chú thích `OfficeStatusCounts là phép chiếu của agent_status (12 -> 6 nhóm)`; SCREEN_SPECIFICATION:19 thêm `(danh sách này là lớp hiển thị; enum gốc 12 trạng thái ở DATABASE_DESIGN §3)`.

### CF-010 run_status vs node_run_status
- Tài liệu: `DATABASE_DESIGN.md:194-196` (`run_status` 7 giá trị dùng chung), `WORKFLOW_ENGINE.md:130-150,394` (cần READY, QUEUED, SKIPPED, WAITING_HUMAN và cột `revision`).
- Bằng chứng: WORKFLOW_ENGINE §5 là state machine chi tiết có chuyển trạng thái; enum DB 7 giá trị không biểu diễn được. `node_runs.revision` chưa có (revision tại DB:916 là của `artifacts`).
- Phương án: A) thêm enum `node_run_status` + cột `node_runs.revision` + unique `(run_id, node_id, revision)` (P: khớp engine; N: thêm migration). B) dùng chung `run_status` mở rộng (N: trộn ngữ nghĩa run và node).
- Khuyến nghị: A, giá trị: `PENDING, READY, QUEUED, RUNNING, WAITING_HUMAN, SUCCEEDED, FAILED, SKIPPED, CANCELLED` (theo WORKFLOW_ENGINE:134-150; đối chiếu lại bảng chuyển trạng thái khi sửa).
- Sửa (SAU D-003): DATABASE_DESIGN:194-196 sau `runStatus` thêm `export const nodeRunStatus = pgEnum('node_run_status', ['PENDING','READY','QUEUED','RUNNING','WAITING_HUMAN','SUCCEEDED','FAILED','SKIPPED','CANCELLED']);`; bảng `node_runs`: `status: nodeRunStatus` thay `runStatus`, thêm `revision: integer('revision').notNull().default(0)` và `uniqueIndex(...).on(runId, nodeId, revision)`.

### CF-011 Case enum task_status
- Tài liệu: `DATABASE_DESIGN.md:186-189,204,1469` (task_status HOA, enum khác thường, tự đánh dấu cần duyệt), DECISION_LOG mục 5, WORKFLOW_ENGINE, ORCHESTRATION M-O8. (Cập nhật: DB đã HOA đúng spec; chỉ còn lệch quy ước giữa các enum.)
- Phương án: A) giữ như hiện tại (task_status và trạng thái workflow HOA theo spec owner; enum khác thường), nhãn hiển thị qua i18n (P: không migration; N: hai quy ước, dễ nhầm). B) thống nhất HOA cho mọi enum trạng thái (P: nhất quán; N: migration, đổi nhiều contract). C) thống nhất thường (N: đổi spec owner).
- Khuyến nghị: A, ghi quy ước vào CODING_STANDARDS.
- Câu hỏi owner: "Giữ `task_status` HOA (theo spec của bạn) và các enum khác chữ thường, hay thống nhất một kiểu cho mọi enum?" (mức ưu tiên thấp; mặc định A.)

### CF-012 provider_kind thiếu giá trị — ALREADY_FIXED
- Xác minh: `DATABASE_DESIGN.md:171-175` nay có `claude_code, codex, openhands, generic_cli, mock, anthropic_api, openai, gemini, ollama, openrouter, custom`.
- Còn lại (không phải mâu thuẫn mới, theo QA P1-09): `mock` vẫn nằm trong enum production và `PROVIDER_INTEGRATION.md:35`. Khuyến nghị tách `mock` ra type của `packages/provider-testkit`.
- Sửa (SAU D-005 và D-006, tùy chọn): DATABASE_DESIGN:174 bỏ `'mock'` khỏi enum và sửa chú thích dòng 172-173; PROVIDER_INTEGRATION:35 `claude_code|codex|openhands|generic_cli|mock` -> `claude_code|codex|openhands|generic_cli` (và định nghĩa `'mock'` trong testkit).

### CF-013 Cổng approval Claude
- Tài liệu: `SECURITY.md:18,93,119,135,351-354,551,565` (PreToolUse HTTP hook, `claude -p` làm nguồn chính) vs D-004/ADR-0004, `PROVIDER_INTEGRATION.md:410,499`, `PERMISSION_MODEL.md` §14. QA P1-05, vẫn còn.
- Phương án: A) hai tầng dùng chung policy engine: `canUseTool` (SDK in-process) là đường hỏi người, PreToolUse hook là lớp cưỡng chế (P: có bằng chứng smoke test canUseTool chặn Write; N: hai tầng có thể trùng quyết định). B) giữ HTTP hook làm chính (P: độc lập SDK; N: giữ kết nối tới 540 s, chưa có bằng chứng chạy trên Windows).
- Khuyến nghị: A.
- Sửa (SAU D-004): SECURITY:119,135,351-354: viết lại §5.1 theo `PROVIDER_INTEGRATION.md §17`: đổi `PreToolUse hook kiểu HTTP ... làm nguồn chính` -> `canUseTool in-process là đường hỏi người; PreToolUse hook là lớp cưỡng chế bắt buộc; cả hai gọi policy engine chung`; :551 M1 `HTTP PreToolUse hook + approvals table` -> `policy engine + canUseTool + PreToolUse hook`; :565 ghi hook type `http` là phương án dự phòng chưa kiểm chứng.

### CF-014 Chế độ spawn Claude
- Tài liệu: `WORKSPACE_AND_GIT_ISOLATION.md:116,129` (`acceptEdits` + file `.claude/settings.json`), `SECURITY.md:94,114` (`manual` + `--settings`), `PROVIDER_INTEGRATION.md:201` (`default` + whitelist).
- Phương án: A) mặc định `default` + whitelist, `acceptEdits/auto` chỉ theo role, settings qua `--settings` (hoặc tùy chọn SDK), không ghi file trong worktree, không `bypassPermissions` ngoài sandbox (P: an toàn, khớp PROVIDER; N: nhiều prompt hơn). B) `acceptEdits` mặc định (P: ít prompt; N: agent tự sửa file hơn, file settings nằm trong vùng agent ghi được).
- Khuyến nghị: A.
- Sửa (SAU D-004): WORKSPACE:116 `"defaultMode": "acceptEdits"` -> `"defaultMode": "default"`; :129 `--permission-mode acceptEdits` -> `--permission-mode default (acceptEdits chỉ theo role)`, bỏ ý ghi `.claude/settings.json` trong worktree, thay bằng `--settings <file ngoài worktree>`; SECURITY:94,114 `--permission-mode manual` -> `--permission-mode default`.

### CF-015 Approval policy Codex
- Tài liệu: `WORKSPACE_AND_GIT_ISOLATION.md:132` (`--ask-for-approval never`) vs `SECURITY.md:98,204` và `PROVIDER_INTEGRATION.md:224` (`on-request`). Cập nhật: SECURITY đã dùng `on-request`; chỉ WORKSPACE còn `never`.
- Phương án: A) app-server `on-request` + `workspace-write`; `never` chỉ cho fallback `codex exec` trong Tier 2 (P: có approval tương tác; N: phụ thuộc API experimental). B) `never` mọi nơi (N: mất cổng duyệt).
- Khuyến nghị: A.
- Sửa (SAU D-005): WORKSPACE:132 `codex exec --sandbox workspace-write --ask-for-approval never --cd <worktree>` -> `codex app-server (approvalPolicy "on-request", sandbox workspace-write); chỉ fallback codex exec (--ask-for-approval never) trong Tier 2`.

### CF-016 Container warm vs ephemeral
- Tài liệu: `SECURITY.md:122,145`, `security/SANDBOX_DESIGN.md:28,271` (hợp nhất warm per project cho dev/test, ephemeral cho preview/custom), `architecture/INFRASTRUCTURE_DESIGN.md:45,94-` (ephemeral).
- Phương án: A) hybrid theo SANDBOX_DESIGN (P: giảm độ trễ khi 18 agent xen kẽ lệnh; N: trạng thái container tích lũy, cần recreate sau install/4 giờ/đổi trust). B) ephemeral toàn bộ (P: sạch; N: chậm, cache node_modules mất).
- Khuyến nghị: A.
- Sửa (SAU D-007): INFRASTRUCTURE_DESIGN:45 `sandbox containers (ephemeral, ...)` -> `sandbox containers (warm per project cho dev/test; ephemeral cho preview/custom; xem SANDBOX_DESIGN §14)`; §6.1 (dòng 94) thêm một câu cùng nội dung; SANDBOX_DESIGN:28 đổi `Đề xuất` thành `Đề xuất (chờ D-007)`.

### CF-017 POL-11/POL-12 vs R030/R040/R050
- Tài liệu: `product/USER_ROLES_AND_PERMISSIONS.md:119,120,148,159` (POL-11 merge: "tùy chọn, code review L3 đủ"; POL-12 dependency: tùy chọn, mặc định tắt) vs `security/PERMISSION_MODEL.md:268,271,276,283` (R030 `merge_main` luôn ask; R040/R050 `dependency_add`), D-009 (không tự merge vào `main`).
- Mâu thuẫn thật, liên quan đến yêu cầu owner "không sửa code người khác khi chưa có phép".
- Phương án: A) merge vào `integration/*` tự động sau Lớp 3 + review L3; vào `main` luôn hỏi owner; `dependency_add` hỏi ở project untrusted, allow + audit ở trusted; Lead không duyệt thay người (P: an toàn; N: nhiều lần hỏi ở main). B) Lead agent duyệt merge vào main (P: ít can thiệp; N: vi phạm D-009, agent tự duyệt). C) mọi merge đều hỏi (P: an toàn nhất; N: chậm).
- Khuyến nghị: A.
- Câu hỏi owner: "Agent được tự merge vào nhánh `integration/*` sau khi qua kiểm diff và review L3, nhưng merge vào `main` luôn cần bạn duyệt; Lead agent không được duyệt thay bạn. Đồng ý? Thêm dependency mới: hỏi bạn ở project chưa tin cậy, cho phép (có ghi audit) ở project tin cậy?"

### CF-018 Engine Office
- Tài liệu: `design/SCREEN_SPECIFICATION.md:142,690` (PixiJS/Canvas [Giả định]) vs D-011, ADR-0011, `VIRTUAL_OFFICE.md` §4. Còn nguyên.
- Khuyến nghị: Phaser 3.90 sau `OfficeRenderer`; ghi lý do giữ 3.x (latest là 4.2.1, QA P2-09).
- Sửa (SAU D-011): SCREEN_SPECIFICATION:142 `Canvas (PixiJS/Canvas 2D — chọn công nghệ thuộc Agent Frontend [Giả định])` -> `Canvas (Phaser 3.90 sau OfficeRenderer, D-011)`; :690 `engine chọn (PixiJS/Canvas)` -> `engine Phaser 3.90 (D-011)`.

### CF-019 Kích thước tile và map
- Tài liệu: `DATABASE_DESIGN.md:1023-1025` (40x24, `tile_size` 32), `design/SCREEN_SPECIFICATION.md:127` ("32px tiles") vs `VIRTUAL_OFFICE.md:60,116,331` (art 16 px, zoom nguyên 2x = 32 px, map 64x40) và Pixel Agents `TILE_SIZE=16` (QA đã xác minh trong mã).
- Bằng chứng: 14 phòng, 40 desk (VIRTUAL_OFFICE:116,331) không vừa 40x24; art nguồn tái dùng là 16 px.
- Khuyến nghị: logic tile = 16 px, `display_scale` 2; default 64x40.
- Sửa (SAU D-011): DATABASE_DESIGN:1023 `.default(40)` -> `.default(64)`; :1024 `.default(24)` -> `.default(40)`; :1025 `tileSize ... .default(32)` -> `tileSize ... .default(16)` và thêm `displayScale: integer('display_scale').notNull().default(2),`; SCREEN_SPECIFICATION:127 `32px tiles` -> `16px art, hiển thị 2x`.

### CF-020 Số animation
- Tài liệu: `DATABASE_DESIGN.md:1079` (comment 7 giá trị có `sleeping`; cột là `text`) vs FR-158 (`product/FUNCTIONAL_REQUIREMENTS.md:168`, 13 animation, [Owner]).
- Bằng chứng: FR-158 là yêu cầu của owner.
- Sửa (SAU D-011): DATABASE_DESIGN:1079 comment `'idle' | 'walk' | 'typing' | 'thinking' | 'talking' | 'sleeping' | 'error'` -> `'idle' | 'walk' | 'run' | 'sit' | 'work' | 'typing' | 'reading' | 'thinking' | 'meeting' | 'talking' | 'celebrating' | 'error' | 'waiting'` (13, theo FR-158). Nếu owner muốn `sleeping`, thêm như animation mở rộng.

### CF-021 Màu blocked
- Tài liệu: `design/SCREEN_SPECIFICATION.md:143` (`blocked/failed` bubble "⛔ đỏ") vs `design/DESIGN_TOKENS.md:58,183` (`blocked` orange) và `VIRTUAL_OFFICE.md:156` (blocked orange, failed đỏ).
- Sửa (SAU D-015): SCREEN_SPECIFICATION:143 `` `blocked/failed` → bubble "⛔" đỏ `` -> `` `blocked` → bubble "⛔" cam (--status-blocked); `failed` → bubble "⛔" đỏ ``.

### CF-022 Relax/Coffee khi idle
- Tài liệu: `design/SCREEN_SPECIFICATION.md:143,678` (idle không đi cà phê, không họp giả) vs FR-151, FR-164 (`product/FUNCTIONAL_REQUIREMENTS.md:161,174`, owner: Relax/Coffee cho agent Idle/Available "chỉ khi có event tương ứng"); `VIRTUAL_OFFICE.md:157-158` đã có policy `coffeeOnCooldown`, `idleToRelax`.
- Phương án: A) phòng luôn có; nhân vật chỉ vào khi event thật (`RATE_LIMITED` hoặc idle kéo dài) và policy bật, mặc định tắt (P: giữ luật "không giả lập tiến độ"; N: văn phòng có vẻ tĩnh). B) bật `idleToRelax` mặc định (P: sinh động; N: gây hiểu nhầm ai đang làm). C) bỏ phòng Relax/Coffee (N: đổi FR-151).
- Khuyến nghị: A.
- Câu hỏi owner: "Agent idle có tự đi tới phòng Relax/Coffee không? Khuyến nghị: tắt mặc định, chỉ bật khi bạn gạt tùy chọn, và chỉ dựa trên sự kiện thật (idle kéo dài hoặc hết quota)."

### CF-023 waiting_approval đứng đâu
- Tài liệu: `SCREEN_SPECIFICATION.md:143` (đi tới Review desk) vs `VIRTUAL_OFFICE.md:150,367` (mặc định đứng desk, tùy chọn `goToReviewDesk`).
- Sửa (SAU D-011): SCREEN_SPECIFICATION:143 `` `waiting_approval` → bubble "✋" + đi tới "Review desk" `` -> `` `waiting_approval` → bubble "✋", đứng tại desk (tùy chọn goToReviewDesk) ``.

### CF-024 OpenHands ACP vs D-008
- Tài liệu: `PROVIDER_INTEGRATION.md:263 (§9.5 #1)` (ACP materialize `CODEX_AUTH_JSON` vào `~/.codex/auth.json` trong container) vs D-008. ADR-0006 đã tắt ACP ở MVP.
- Phương án: A) tắt ACP ở MVP, chỉ OpenHands native + LLM key trong keychain (P: giữ D-008; N: OpenHands cần API key riêng, tốn chi phí token). B) duyệt ACP (P: dùng login CLI; N: phá D-008, `auth.json` lọt vào container). 
- Khuyến nghị: A.
- Câu hỏi owner: "Giữ chế độ ACP của OpenHands tắt ở MVP (đồng nghĩa OpenHands dùng API key riêng lưu trong keychain)? Hay chấp nhận cho OpenHands đọc `auth.json` của Codex?"

### CF-025 Giấy phép sprite JIK-A-4
- Tài liệu: `research/PIXEL_AGENTS_ANALYSIS.md:16,366,442,491` (chưa xác minh) vs `research/LICENSE_AND_REUSE.md:46,210-211,352` và `VIRTUAL_OFFICE.md:271` (CC0 "đã xác minh").
- Xác minh: không có bằng chứng lưu trong repo; WebFetch tới itch.io trả 403 trong phiên này, nên chưa kiểm được.
- Khuyến nghị: nhãn tài liệu phải phản ánh mức bằng chứng thật; không bundle asset cho tới khi có ảnh chụp trang license (không lấy qua bản Pixel Agents).
- Sửa (NGAY): LICENSE_AND_REUSE:352 `**CC0 1.0** (hiển thị trên trang)` -> `**CC0 1.0 [chưa lưu bằng chứng: cần ảnh chụp trang itch.io trong PR]**`; VIRTUAL_OFFICE:271 thêm `[Giả định tới khi có ảnh chụp license]`; LICENSE_AND_REUSE:46 giữ nguyên ý "chưa xác minh". Việc cần người: một thành viên mở trang itch.io và lưu ảnh chụp vào `docs/research/evidence/` (thư mục mới, Phase 1).

### CF-026 CC0-only vs D-012
- Tài liệu: yêu cầu gốc "asset chỉ CC0" vs D-012 (CC0/MIT/OFL), `research/LICENSE_AND_REUSE.md:134` (CC0 hoặc MIT/CC-BY có file license), `VIRTUAL_OFFICE.md` §21 #1. D-012 nói OFL còn LICENSE_AND_REUSE nói CC-BY: hai danh sách khác nhau.
- Phương án: A) CC0 mặc định; MIT/OFL cho phép kèm THIRD_PARTY_NOTICES; CC-BY chỉ khi owner duyệt từng gói (P: pháp lý sạch; N: ít lựa chọn asset). B) chỉ CC0 tuyệt đối (P: đúng yêu cầu gốc; N: hạn chế font/icon). C) cho phép CC-BY chung (N: cần credit, rủi ro quên).
- Khuyến nghị: A.
- Câu hỏi owner: "Yêu cầu gốc là asset chỉ CC0. Cho phép thêm MIT và OFL (font) kèm file license và THIRD_PARTY_NOTICES; CC-BY chỉ khi bạn duyệt từng gói?"

### CF-027 Turborepo
- Tài liệu: `development/CI_CD.md:11,20,46,59,78,86,113,135,151` (pnpm + Turborepo, [Cần phê duyệt]) vs D-001/ADR-0001 (pnpm thuần, Turborepo sau).
- Khuyến nghị: pnpm workspaces thuần; thêm Turborepo khi > ~5 package hoặc CI chậm (ADR-0001). Mức rủi ro thấp.
- Sửa (SAU D-001): CI_CD:11,20 `pnpm 10 + Turborepo [Cần người dùng phê duyệt]` -> `pnpm 10 workspaces (Turborepo thêm sau khi > ~5 package hoặc CI chậm, ADR-0001)`; :46 `Turborepo --filter=...[origin/main]` -> `pnpm --filter "...[origin/main]"`; :59 xóa `TURBO_TELEMETRY_DISABLED: 1`; :78,86,113,135,151 `pnpm turbo <task>` -> `pnpm -r --if-present run <task>`.

### CF-028 Test runner
- Tài liệu: `research/CHATDEV_ANALYSIS.md:571,648` (Jest), `architecture/AGENT_ORGANIZATION.md:41` (skill `jest`), `WORK_PLAN.md:47` ("vitest/jest") vs `TECHNOLOGY_EVALUATION.md:236-243` (Vitest; api Vitest + `unplugin-swc` nếu spike qua, nếu không Jest).
- Khuyến nghị: Vitest cho `apps/web` và `packages/*`; api quyết định bằng spike nhỏ ở P1 (đây là việc kỹ thuật, không cần owner; nếu spike thất bại, api dùng Jest, một runner mỗi package).
- Sửa (SAU D-001): CHATDEV_ANALYSIS:648 `Port sang TS/Jest` -> `Port sang TS/Vitest`; :571 `Jest` -> `Vitest`; AGENT_ORGANIZATION:41 skill `jest` -> `vitest`; WORK_PLAN:47 `vitest/jest` -> `Vitest (api: spike unplugin-swc, dự phòng Jest)`.

### CF-029 Số loại node workflow
- Tài liệu: FR-113 (`product/FUNCTIONAL_REQUIREMENTS.md:133`, 8 loại, P2, [Owner]), yêu cầu hiện tại 9 (có `Task`), `WORKFLOW_ENGINE.md:423 (M-W2)` 11 `type`, `SCREEN_SPECIFICATION.md:471` 9 node type, DB comment `agent_task`/`join`.
- Phương án: A) 11 `kind` nội bộ trong engine, palette UI 9 loại theo yêu cầu (P: engine đủ cho Task/Join; N: ánh xạ UI-engine). B) chỉ 8 loại FR-113 (P: đúng FR cũ; N: thiếu Task). C) 9 loại cả engine (P: đơn giản; N: thiếu join/merge nội bộ).
- Khuyến nghị: A.
- Câu hỏi owner: "FR-113 nói 8 loại node, yêu cầu mới nói 9 (thêm Task). Bạn chấp nhận palette UI 9 loại (cập nhật FR-113) trong khi engine nội bộ có 11 kind?"

### CF-030 Template mặc định: stage và auto-deploy
- Tài liệu: `research/ORCHESTRATION_WORKFLOW_RESEARCH.md:651,729-734` (8 stage khác, node `Release Production`) vs FR-119..127 (`FUNCTIONAL_REQUIREMENTS.md:139-147`, [Owner], không auto-deploy production) và `WORKFLOW_ENGINE.md:243-247,430 (M-W9)` đã theo FR.
- Bằng chứng: FR-119..127 do owner đặt; AGENTS.md và FR-126/FR-199 cấm auto-deploy production. WORKFLOW_ENGINE đã theo FR; chỉ research A07 lỗi thời.
- Sửa (NGAY): ORCHESTRATION_WORKFLOW_RESEARCH:651 thêm dòng đầu `[LỖI THỜI] Template chính thức theo WORKFLOW_ENGINE.md §10 (FR-119..127); không auto-deploy production.`; :734 `"target": "production"` kèm nhãn `minh họa, bị cấm bởi FR-126`.

### CF-031 PM chấm điểm
- Tài liệu: FR-109 (`FUNCTIONAL_REQUIREMENTS.md:129`, "PM agent chấm điểm", [Owner]) vs `MULTI_AGENT_ORCHESTRATION.md:131` và AGENT_ARCHITECTURE A13 §5.3 (scoring xác định bằng mã, PM chỉ xử lý bế tắc).
- Phương án: A) scoring xác định bằng mã; PM agent chỉ can thiệp khi bế tắc (P: miễn phí token, tái lập được; N: khác chữ FR-109). B) PM agent chấm điểm mọi task (P: đúng FR; N: tốn token và quota cho mỗi task, kết quả không tái lập). C) lai: mã lọc ứng viên, PM chọn trong top-3 (P: cân bằng; N: vẫn tốn token).
- Khuyến nghị: A.
- Câu hỏi owner: "FR-109 nói PM agent chấm điểm chọn agent. Đồng ý đổi thành: hệ thống chấm điểm bằng công thức xác định (không tốn token), PM agent chỉ can thiệp khi không chọn được?"

### CF-032 Tên branch/worktree
- Tài liệu: `MULTI_AGENT_ORCHESTRATION.md:326 (M-O6)` đã chọn D-009; `research/ORCHESTRATION_WORKFLOW_RESEARCH.md:88,701` (`wf/{{run.id}}/integration`); `development/DEVELOPMENT_GUIDE.md:84` (`agent/<role>/<task-id>`, role = slug, chấp nhận được).
- Sửa (SAU D-009): ORCHESTRATION_WORKFLOW_RESEARCH:701 `"into": "wf/{{run.id}}/integration"` -> `"into": "integration/<project>"`; :88 chú thích `worktree/branch theo D-009: agent/<slug>/<task-id>`.

### CF-033 Vị trí DECISION_LOG — ALREADY_FIXED
- Xác minh: không có liên kết `docs/DECISION_LOG.md` nào còn trong docs (grep); chỉ `research/ARCHITECTURE_COMPARISON.md:134 (C-7)` mô tả mâu thuẫn. ADR README trỏ về file gốc. Không cần sửa; chấp nhận C-7 là bản ghi lịch sử.

### CF-034 Đường dẫn SECURITY.md và DATABASE_DESIGN.md
- Tài liệu (grep, còn nguyên): dòng 5 của `architecture/INFRASTRUCTURE_DESIGN.md`, `architecture/WORKSPACE_AND_GIT_ISOLATION.md`, `development/CI_CD.md`, `development/DEPLOYMENT.md`, `development/ENVIRONMENT_SETUP.md`, `operations/DEVSECOPS_GUIDE.md`; `security/SANDBOX_DESIGN.md:306` trỏ file không tồn tại `security/VERIFICATION_LOG.md`.
- Sửa (NGAY): tại dòng 5 của sáu file trên, `docs/security/SECURITY.md` -> `docs/SECURITY.md`; tại dòng 5 của INFRASTRUCTURE_DESIGN, WORKSPACE_AND_GIT_ISOLATION, DEPLOYMENT, DEVSECOPS_GUIDE, `docs/architecture/DATABASE_DESIGN.md` -> `docs/DATABASE_DESIGN.md`; SANDBOX_DESIGN:306 `docs/security/VERIFICATION_LOG.md` -> `docs/security/SANDBOX_VERIFICATION_LOG.md [file sẽ tạo ở Phase 1; chưa tồn tại]`.

### CF-035 Thiếu bảng event_outbox
- Theo kết luận CF-004 (phương án A): thêm bảng.
- Sửa (SAU D-014): DATABASE_DESIGN §4.14 (cạnh `agent_events`): thêm `export const eventOutbox = pgTable('event_outbox', { eventId: uuid('event_id').primaryKey(), globalSeq: bigint('global_seq', { mode: 'number' }).notNull(), createdAt: timestamp... }, (t) => [index('event_outbox_seq_idx').on(t.globalSeq)])`; đổi mọi chỗ "44 bảng" -> "45 bảng" (DATABASE_DESIGN, DECISION_LOG D-002, ADR-0002); :1231 `Hai phương án: (a)... (b)...` -> `Quyết định: outbox ở mọi chế độ (EVENT_SYSTEM §7.1)`; :1471 đóng câu hỏi mở.

### CF-036 Thiếu bảng template version và bảng phụ
- Tài liệu: `AGENT_ARCHITECTURE.md` §3.1.1 (cần `agent_templates`, `agent_template_versions`, skills kèm trọng số), `design/HERDR_DESIGN_INTEGRATION.md` §2 (`notification_reads`), DB chỉ có `is_template` (`DATABASE_DESIGN.md:354,554`).
- Sửa (SAU D-016 và D-017): DATABASE_DESIGN thêm bảng `agent_templates(id, organization_id, code, name, ...)`, `agent_template_versions(id, template_id, version, config jsonb, skills jsonb, UNIQUE(template_id, version))`, `notification_reads(user_or_client_id, entity_type, entity_id, seen_at)`; `agents` thêm cột `template_version_id`; bảng tổng 44 -> 48 (CF-035 thêm 1 và CF-036 thêm 3); cập nhật D-002 mô tả khi owner duyệt.

### CF-037 Thiếu AC cho nhóm FR MVP
- Tài liệu: `IMPLEMENTATION_ROADMAP.md` §7 mục 11, `product/ACCEPTANCE_CRITERIA.md` (100 AC, thiếu dải 016-020, 035-040, 065-070, 083-085, 097-100, 115-120), QA P0-1 (50/199 FR không có AC; 31 FR Must/MVP).
- Đây là việc viết tài liệu, không phải một chỉnh sửa cơ học. Phạm vi: viết AC cho FR-002,005,006,009,010,011; FR-181..187; FR-199; FR-233; FR-236,238; FR-241; FR-261..284.
- Sửa (NGAY, tác giả BA): ACCEPTANCE_CRITERIA thêm AC-101..AC-1xx cho nhóm trên theo cột `FR`; cập nhật `development/TESTING.md:11` `AC-001…AC-130` -> dải thực tế; ghi các dải bỏ trống là dự trữ.

### CF-038 Số lần retry tối đa
- Tài liệu: `AGENT_ARCHITECTURE.md:281 (M-08)`, `architecture/AGENT_SCHEDULER.md:50,186` (`maxRetries`), `DATABASE_DESIGN.md:350,681` (`max_retry` 2), `MULTI_AGENT_ORCHESTRATION.md:153-159`.
- Bằng chứng: `maxAttempts=3` (A07) = lần chạy tổng gồm lần đầu; `max_retry=2` = số lần thử lại, hai cách đếm này khớp (3 attempt). Chỉ A13 `maxRetries=3` (4 attempt) lệch. Mỗi retry tốn một spawn (~48k token, RESOURCE_MANAGEMENT:212).
- Khuyến nghị: `max_retry = 2` (3 attempt tổng), đọc từ `tasks.max_retry`, sau đó handover.
- Sửa (SAU D-010): AGENT_SCHEDULER:50 chú thích `maxRetries (= tasks.max_retry, mặc định 2 = 3 attempt tổng)`; :186 `attempt > maxRetries` giữ nhưng thêm định nghĩa `attempt` bắt đầu từ 0; AGENT_ARCHITECTURE:281 `Giá trị cuối lấy từ task` thêm `mặc định 2; maxAttempts=3 tương đương`.

### CF-039 Fallback giữa provider
- Tài liệu: `DATABASE_DESIGN.md:383,391` (`agent_provider_connections.priority` 0 = primary, 1.. = fallback), `AGENT_ARCHITECTURE.md:279 (M-06)`, `architecture/AGENT_ORGANIZATION.md:203` (OI cần phê duyệt), RESOURCE_MANAGEMENT (không auto-switch).
- Phương án: A) không auto-failover; `priority` chỉ gợi ý để owner chuyển thủ công (P: không đốt quota nhà cung cấp khác, không đổi hành vi/an ninh âm thầm; N: task dừng khi provider hết quota). B) auto-failover Claude -> Codex (P: liên tục; N: chi phí bất ngờ, hai quyền/sandbox khác nhau, khó audit). 
- Khuyến nghị: A.
- Câu hỏi owner: "Khi một provider hết quota, hệ thống chỉ báo và chờ bạn chuyển (khuyến nghị), hay tự chuyển task sang provider khác?"

### CF-040 Tên interface adapter
- Tài liệu: `research/CLAUDE_CODE_INTEGRATION.md:535,589,605` (`BrainAdapter`), `research/CODEX_INTEGRATION.md:557,559,563` (`AgentBrain`, `AgentEvent`) vs `BACKEND_ARCHITECTURE.md:141,174 (B-5)` và `PROVIDER_INTEGRATION.md:472 (P1)` (`AgentAdapter`/`AgentEvent`).
- Sửa (NGAY): CLAUDE_CODE_INTEGRATION:535 thêm dưới tiêu đề `Ghi chú: BrainAdapter/BrainEvent là tên nghiên cứu; tên chính thức là AgentAdapter/AgentEvent (PROVIDER_INTEGRATION §P1, packages/agent-core).`; CODEX_INTEGRATION:559 thêm ghi chú tương tự cho `AgentBrain`.

### CF-041 Xác thực
- Tài liệu: `DATABASE_DESIGN.md:1267` (JWT/session) vs `SECURITY.md:461-468` (bearer per-launch) và `BACKEND_ARCHITECTURE.md:41` (desktop bearer, server JWT/session) đã hòa giải.
- Sửa (SAU D-001): DATABASE_DESIGN:1267 `set từ JWT/session` -> `set từ bearer token per-launch (desktop) hoặc JWT/session (server mode), BACKEND_ARCHITECTURE §3.1`.

### CF-042 Quy ước đặt tên hạ tầng
- Tài liệu: `research/CODEX_INTEGRATION.md:620,648,686` (`apps/backend`), `research/ORCHESTRATION_WORKFLOW_RESEARCH.md:219-251,578` (`workspaceId`), `EVENT_SYSTEM.md:238,397,406,409`, `SYSTEM_ARCHITECTURE.md:300-306`.
- Khuyến nghị v1 giữ nguyên, với một điều chỉnh: UUID v7 cần hàm ứng dụng hoặc PostgreSQL 18; ADR-0002 chọn PG 16 và DATABASE_DESIGN dùng `gen_random_uuid()` (v4). Mặc định: UUID v4 + sắp theo `global_seq`/`created_at`; v7 là tùy chọn sau.
- Sửa (SAU D-001 và D-014): CODEX_INTEGRATION:620,648,686 `apps/backend/src/agents/codex/` -> `apps/api/src/agents/codex/`; ORCHESTRATION_WORKFLOW_RESEARCH:219,238,578 `workspaceId` -> `organizationId` (và `@@index([workspaceId...` tương ứng); EVENT_SYSTEM:406 `UUID v7 (hoặc v4 + global_seq)` -> `UUID v4 (gen_random_uuid) + global_seq; v7 tùy chọn khi PG 18`; :397 đóng câu hỏi mở.

### CF-043 CLI trên host vs "không chạy code AI trên host"
- Tài liệu: DECISION_LOG mâu thuẫn 1, `SECURITY.md` §10 (R1-R10), `SYSTEM_ARCHITECTURE.md` §6, `PROJECT_RISKS.md`, ADR-0007.
- Phương án: A) D-007 hai tầng và owner ký R1, R2, R3, R5, R6 (P: giữ được đăng nhập OAuth của CLI; N: không có ranh giới OS-level cho CLI, token đọc được bởi process cùng user). B) Tier 2 (container hóa hoàn toàn) làm mặc định (P: đúng yêu cầu gốc; N: cần API key riêng, mất subscription login, tốn chi phí token, OpenHands). C) hoãn đến sau Phase 4 (N: chặn thiết kế sandbox).
- Khuyến nghị: A, kèm điều kiện: Docker fail-closed và test SB-00..SB-17 (CF-045).
- Câu hỏi owner: xem D-007 và Q tương ứng trong register (ký chấp nhận R1, R2, R3, R5, R6).

### CF-044 18 agent vs quota và điều khoản dịch vụ
- Tài liệu: DECISION_LOG mâu thuẫn 2, `security/THREAT_MODEL.md` §7, `PROJECT_RISKS.md` R-TOS-QUOTA, ADR-0010, ADR-0004. Chưa có bằng chứng ToS (QA §7).
- Phương án: A) mặc định 5, owner tự xác minh gói và điều khoản, hiển thị gauge quota (P: an toàn; N: owner phải làm việc). B) mặc định 18 (N: nguy cơ cạn quota và vi phạm ToS). C) API key trả theo token (P: không dính ToS subscription; N: chi phí cao hơn, đổi D-004/D-008).
- Khuyến nghị: A.
- Câu hỏi owner: "Bạn xác nhận gói Claude (hiện là `team`) và ChatGPT cho phép chạy nhiều phiên song song theo điều khoản? Hạn mức chấp nhận: 5 agent mặc định, nâng lên 10/18 chỉ sau khi đo thực?"

### CF-045 Khoảng trống chưa kiểm chứng (Docker, Codex)
- Xác minh hôm nay: `docker info` vẫn báo daemon không chạy (không có pipe `dockerDesktopLinuxEngine`); `where codex` không thấy. Mọi test SB-00..SB-17 và live test Codex chưa chạy.
- Việc này cần owner thao tác trên máy (bật Docker Desktop, cài Codex tốn quota) hoặc quyết định thu hẹp phạm vi, nên là OWNER_DECISION.
- Phương án: A) owner bật Docker Desktop và cài `@openai/codex` trước Phase 4, chạy SB-00..SB-17 (P: dữ liệu thật; N: tốn thời gian và quota). B) hoãn Codex khỏi MVP, chỉ Claude + Docker (P: ít rủi ro; N: mất provider thứ hai). C) hoãn cả hai tới Phase 4 (N: thiết kế sandbox tiếp tục dựa giả định).
- Khuyến nghị: A cho Docker (chặn D-007), B/A cho Codex tùy ngân sách quota.
- Câu hỏi owner: "Bạn sẽ bật Docker Desktop và cài Codex CLI để chạy kiểm chứng trước Phase 4 chứ? Nếu không, Codex ra khỏi MVP?"

### CF-046 Schema manifest quyền file
- Tài liệu: `architecture/WORKSPACE_AND_GIT_ISOLATION.md:92,103,144,231,241` (JSON, `readOnly`) vs `SECURITY.md:428-432` (YAML, `readonly`, key `forbidden`).
- Sửa (SAU D-009): SECURITY:431 `readonly:  ["src/**", ...]` -> `"readOnly": [...]`; chuyển khối :429-432 từ YAML sang JSON `{ "schemaVersion": 1, "writable": [...], "readOnly": [...], "forbidden": [...] }` và thêm ghi chú `manifest lưu ngoài worktree (WORKSPACE_AND_GIT_ISOLATION §3)`.

### CF-047 Cấu hình/env của Claude
- Tài liệu: `architecture/RESOURCE_MANAGEMENT.md:70,204` (`CLAUDE_CONFIG_DIR` riêng), research Claude (`settingSources`), `operations/DEVSECOPS_GUIDE.md:141` (env whitelist thiếu `APPDATA/LOCALAPPDATA`).
- Bằng chứng (lệnh chạy 2026-10-30, không đọc credential): `claude auth status` mặc định `loggedIn:true`; với `CLAUDE_CONFIG_DIR` trỏ thư mục rỗng thì `loggedIn:false, authMethod:none`. Vậy `CLAUDE_CONFIG_DIR` riêng làm mất đăng nhập OAuth.
- Khuyến nghị: không dùng `CLAUDE_CONFIG_DIR` riêng; dùng `settingSources` và `--settings`; thêm `APPDATA`, `LOCALAPPDATA` vào env whitelist.
- Sửa (SAU D-004): RESOURCE_MANAGEMENT:70 `CLAUDE_CONFIG_DIR=<app-managed dir> chứa settings.json riêng ..., hoặc dùng flag settings/` -> `Không dùng CLAUDE_CONFIG_DIR riêng (đã kiểm chứng: mất đăng nhập OAuth). Dùng settingSources + --settings/--strict-mcp-config`; DEVSECOPS_GUIDE:141 `` `CLAUDE_CONFIG_DIR` nếu dùng `` -> `APPDATA`, `LOCALAPPDATA`` (xóa `CLAUDE_CONFIG_DIR`); SECURITY env whitelist tương tự.

### CF-048 WAITING_APPROVAL có chiếm slot
- Tài liệu: `RESOURCE_MANAGEMENT.md:55,104` (không chiếm slot, quá 24 h về QUEUED) vs `PROVIDER_INTEGRATION.md:411` (approval mức tool giữ process sống nên chiếm slot).
- Sửa (SAU D-010): RESOURCE_MANAGEMENT:55 `PAUSED và WAITING_APPROVAL không chiếm slot (process đã dừng)` -> `PAUSED không chiếm slot; WAITING_APPROVAL mức tool chiếm slot cho tới ngưỡng chờ (CF-007) rồi chuyển WAITING_HUMAN (không chiếm); approval mức task không chiếm slot`; :104 `WAITING_APPROVAL quá hạn → task về QUEUED` -> `quá ngưỡng chờ → task WAITING_HUMAN (CF-007)`.

### CF-049 API đọc quota
- Tài liệu: `RESOURCE_MANAGEMENT.md:107-118,219` (chưa có API; Codex giả định) vs `PROVIDER_INTEGRATION.md:98,155,183` (`rate_limit_event.unifiedWindows`, `account/rateLimits/read`).
- Mức bằng chứng: tài liệu, chưa chạy live (QA §7).
- Sửa (NGAY): RESOURCE_MANAGEMENT §8.1 bullet đầu (`[Giả định — chưa xác minh bằng tài liệu...]`) thêm cuối `Nguồn đọc quota: rate_limit_event.unifiedWindows (PROVIDER_INTEGRATION §7) [Tài liệu, chưa chạy live]`; §8.2 `[Giả định — CLI chưa cài]` thêm `account/rateLimits/read (bindings app-server 0.164)`; :219 câu hỏi 2 đánh dấu `(đã có đường đọc: xem §8.1, 8.2)`.

### CF-050 Tên event approval Codex
- Tài liệu: `SECURITY.md:207,356` (`exec_approval_request`, `apply_patch_approval_request`) vs `PROVIDER_INTEGRATION.md:505` (v2: `item/commandExecution/requestApproval`).
- Sửa (SAU D-005): SECURITY:207 và :356 `exec_approval_request` -> `item/commandExecution/requestApproval`; `apply_patch_approval_request` -> `item/fileChange/requestApproval` (đối chiếu bindings `codex app-server generate-ts` trước khi áp dụng).

### CF-051 Hash chain audit và phạm vi
- Tài liệu: `operations/DEVSECOPS_GUIDE.md:123` ("hash chain tùy chọn sau MVP") vs `SECURITY.md:496` và `security/AUDIT_LOGGING.md` §7 (bắt buộc, từ M0); `SECURITY.md:488` category `tool_call/tool_result` vào audit vs AUDIT_LOGGING:15 (tách `agent_events` hoạt động và `audit_logs` quyết định).
- Khuyến nghị: theo AUDIT_LOGGING (chi phí thấp, bật từ M0); tool_call/tool_result chi tiết ở `agent_events`, audit chỉ ghi quyết định và cưỡng chế, nối bằng `correlation_id`.
- Sửa (SAU D-008): DEVSECOPS_GUIDE:123 `Hash chain (prev_hash) tuỳ chọn sau MVP để phát hiện sửa đổi.` -> `Hash chain (prev_hash, hash) bắt buộc từ M0 (AUDIT_LOGGING §7).`; SECURITY:488 category `tool_call|tool_result` -> `tool_decision` (quyết định allow/deny của tool call) và thêm `tool_call/tool_result chi tiết lưu ở agent_events, liên kết bằng correlation_id`.

### CF-052 Bảng secrets vs credential_ref
- Tài liệu: `SECURITY.md:297` (bảng `secrets(id, name, provider, keychain_ref, ...)`), `operations/DEVSECOPS_GUIDE.md:174`, `security/CREDENTIAL_MANAGEMENT.md:60,210` vs `DATABASE_DESIGN.md:320,877,1269` (`credential_ref`).
- Khuyến nghị (D-008): chỉ `credential_ref` (khóa keychain) ở MVP, không bảng `secrets`.
- Sửa (SAU D-008): SECURITY:297 `Postgres bảng secrets(id, name, provider, keychain_ref, created_at, last_used_at) — không có cột value.` -> `Postgres chỉ lưu credential_ref (khóa keychain) trong provider_configs/git_repositories (DATABASE_DESIGN §4); không có bảng secrets ở MVP.`; DEVSECOPS_GUIDE:174 `schema secrets` -> `credential_ref`; CREDENTIAL_MANAGEMENT:210 đánh dấu đã chốt.

### CF-053 Worktree host-truth vs volume-truth
- Tài liệu: `security/SANDBOX_DESIGN.md:16,48-49,93-94,145` (named volume `aiw-<pid>-ws`) vs D-007/D-009 và `architecture/WORKSPACE_AND_GIT_ISOLATION.md` (worktree trên host).
- Bằng chứng: Tier 1 chạy CLI trên host sửa file trong worktree host (D-007), nên nguồn sự thật phải ở host; volume làm nguồn sự thật bắt buộc đồng bộ ngược về worktree. Volume chỉ hợp cho `node_modules` (hiệu năng 9P) và cho Tier 2.
- Khuyến nghị: host-truth cho Tier 1 (bind chỉ phần `writable`), `node_modules` named volume, Tier 2 dùng volume rồi export diff. Hiệu năng 9P chưa đo (Docker chưa chạy).
- Sửa (SAU D-007 và D-009): SANDBOX_DESIGN:48 `--mount type=volume,src=aiw-<projectId>-ws,dst=/workspace` -> `--mount type=bind,src=<worktree host>,dst=/workspace` (chỉ phần writable); giữ :49 volume `node_modules`; :93 và :145 tương ứng; :16 xóa `Ngoài: ... host-truth vs volume` và ghi `Đã chốt: host-truth cho Tier 1 (CF-053)`.

### CF-054 Phạm vi tái dùng UI OpenHands
- Tài liệu: `research/LICENSE_AND_REUSE.md:122,205` (ALLOW UI component, hooks) vs `research/OPENHANDS_ANALYSIS.md` §11.3 (UI gắn chặt HeroUI/i18n/PostHog/Cloud).
- Sửa (SAU D-012 và D-006): LICENSE_AND_REUSE:122 `UI component, hooks, MSW mocks, Playwright config` -> `MSW mocks, Playwright config (UI component và hooks: chỉ tham khảo, gắn chặt HeroUI/PostHog/Cloud)`; :205 `UI components/hooks.` -> `Electron bootstrap patterns (UI là reference-only).`

### CF-055 Vai trò duyệt workflow vs RBAC
- Tài liệu: `WORKFLOW_ENGINE.md:95,202,426 (M-W5)` (`approverRoles` product/tech_lead/qa_lead) vs DB `org_role` (owner/admin/member/viewer).
- Bằng chứng: MVP một người dùng (SKILL_SYSTEM:497: mọi quyền = owner); `approverRoles` là nhãn quyền. Bảng ánh xạ đầy đủ cho multi-user hoãn tới P3.
- Sửa (SAU D-003): WORKFLOW_ENGINE:95 `Nhãn quyền; phải ánh xạ được sang org_role (M-W5)` -> `Nhãn quyền; MVP (một người dùng) mọi nhãn ánh xạ tới owner; bảng ánh xạ multi-user chốt trước P3 cùng USER_ROLES_AND_PERMISSIONS`; :426 đổi cột xử lý thành `MVP: tất cả -> owner; P3: bảng ánh xạ`.

## 3. Việc chặn bởi từng nhóm

Các `OWNER_DECISION` nên được gom theo chủ đề trong `PHASE_0_OWNER_DECISION_REGISTER.md` §4 (Q-01..Q-15). Các `STALE_DOC_SYNC` có điều kiện `SAU D-xxx` được giải phóng khi D-xxx tương ứng được owner ghi APPROVED; điều kiện `NGAY` có thể áp dụng bất cứ lúc nào (CF-005, CF-025, CF-030, CF-034, CF-037, CF-040, CF-049).

---

# PHIÊN BẢN 1 (nguyên văn, System Architect)

# CONFLICTS_TO_RESOLVE — Mâu thuẫn giữa các tài liệu Phase 0

Ngày: 2026-10-30. Tổng hợp và khử trùng lặp từ: `docs/SYSTEM_ARCHITECTURE.md` §8, `AGENT_ARCHITECTURE.md` §8, `MULTI_AGENT_ORCHESTRATION.md` §15, `WORKFLOW_ENGINE.md` §15, `EVENT_SYSTEM.md` §14, `PROVIDER_INTEGRATION.md` §25, `VIRTUAL_OFFICE.md` §21, `docs/security/*.md` (mục Vấn đề chưa giải quyết/§12), `research/ARCHITECTURE_COMPARISON.md` §6.3, `PROJECT_RISKS.md`, `IMPLEMENTATION_ROADMAP.md` §3.14, `BACKEND_ARCHITECTURE.md`, `DECISION_LOG.md`.

Mọi cột "Đề xuất xử lý" là đề xuất của System Architect, chưa được duyệt. Cột "Cần owner duyệt": **Có** = cần chủ sở hữu chọn hoặc chấp nhận trước khi đóng băng contract/triển khai; **Không** = sửa tài liệu theo quyết định đã nêu, đi theo ADR liên quan.

| ID | Chủ đề | Tài liệu và giá trị mâu thuẫn | Đề xuất xử lý | Liên quan | Cần owner duyệt |
|---|---|---|---|---|---|
| CF-001 | ORM: Prisma vs Drizzle | ORCHESTRATION_WORKFLOW_RESEARCH §3.2 (schema và `prisma.$transaction`), AGENT_ORGANIZATION (`prisma/**`), WORK_PLAN §2 ("Prisma/Drizzle"), WORKFLOW_ENGINE M-W10, ORCHESTRATION M-O5 vs DATABASE_DESIGN §1 và D-002 (Drizzle) | Dùng Drizzle; mã Prisma trong tài liệu orchestration chỉ minh họa; đổi `prisma/**` thành `packages/db/**`; chạy spike partition 1 ngày trước khi khóa | ADR-0002 / D-002 | Có |
| CF-002 | Concurrency mặc định | ORCHESTRATION §0 và A07: global 10, per-provider 6; CLAUDE_CODE_INTEGRATION §6: `MAX_PARALLEL_AGENTS` 10; CODEX_INTEGRATION §9: 2–3 turn/account; DB `provider_configs.max_concurrent` mặc định 2; AGENT_ARCHITECTURE A13: claude 5, codex 3; D-010: mặc định 5, tối đa 18 | Global mặc định 5, trần 18, cap provider qua `ResourceConfig` (Claude 5, Codex 3); mặc định DB 2 chỉ là giá trị khởi tạo; nâng 5 → 10 → 18 theo bằng chứng | ADR-0010 / D-010 | Có |
| CF-003 | Quy ước tên event | ORCHESTRATION §7 và A07: UPPER_SNAKE (`AGENT_CREATED`, `NODE_*`); DATABASE_DESIGN §5.2: dotted lowercase (`session.started`); NAVIGATION §10: `approval.requested`; EVENT_SYSTEM M-E2 chọn UPPER_SNAKE trong khi SYSTEM_ARCHITECTURE §8 #3 chọn dotted lowercase; VIRTUAL_OFFICE §21 #9: `agent.state_changed` vs `AGENT_STATE_CHANGED` | `type` lưu dotted lowercase; một bảng ánh xạ 1-1 sang UPPER_SNAKE ở lớp hiển thị, đặt trong `@aiw/contracts` (API_DOCUMENTATION §6.3); ghi vào DECISION_LOG | ADR-0014 / D-014 | Có |
| CF-004 | Outbox vs event store append-only | A07: `EventOutbox.published` + SKIP LOCKED nhiều relay, replay 7 ngày; DATABASE_DESIGN §5.4: hai chế độ (a)/(b), retention 90/365 ngày; D-002: event store append-only (không UPDATE) | Một relay duy nhất đọc `agent_events` theo con trỏ `global_seq` (bảng cursor nhỏ), giữ append-only, không cột `published`; consumer idempotent theo `event.id` | ADR-0014, ADR-0002 | Có |
| CF-005 | Tên bảng audit | DATABASE_DESIGN và AUDIT_LOGGING: `audit_logs`; SECURITY, DEVSECOPS, INFRASTRUCTURE_DESIGN: `audit_events` | Giữ `audit_logs` (đã trong 44 bảng); áp migration AUDIT_LOGGING §4.3 (hash chain, `category`, `input_hash`); sửa các tài liệu còn lại | D-002, D-008 | Có |
| CF-006 | Retention audit | SECURITY: 180 ngày; DEVSECOPS: 365 ngày (EVENT_SYSTEM: 90/365 cho event) | 365 ngày; purge theo quy trình export, verify, anchor, detach (AUDIT_LOGGING AU-10) | D-002 | Có |
| CF-007 | Timeout approval | SECURITY §5.3 và BACKEND: hook HTTP 540 s rồi deny; DATABASE_DESIGN: `timeout_sec` 3600, `onTimeout reject`; USER_ROLES §5.5: không timeout, nhắc 24 h; PROVIDER_INTEGRATION §17: 10 phút; research Claude: 30 phút cho `canUseTool` | Hook trả deny `pending_human` sau 540 s, approval vẫn `pending`, task `WAITING_HUMAN`, kích hoạt lại khi duyệt (PERMISSION_MODEL §6.4); `prod_deploy/data_deletion/merge_main` luôn `scope=once`, ttl ≤ 15 phút | ADR-0003, ADR-0004 | Có |
| CF-008 | Mức quyền L0–L5 vs enum | AGENT_ORGANIZATION: 5 level; AGENT_ARCHITECTURE A13: L0..L4; PERMISSION_MODEL: L0–L5; DB `permission_level`: 4 giá trị; `full` vô nghĩa | Ánh xạ L0–L5 sang enum + profile; cấm `full` bằng CHECK constraint; chốt số level trước P1-04 | ADR-0016, ADR-0007 | Có |
| CF-009 | `agent_status`: 12 lifecycle vs 5 event vs 8 tone | A07: 5 giá trị (`idle/working/waiting/error/offline`); DB và PRD FR-024: 12; SCREEN_SPEC §2 danh sách riêng; StatusBadge 8 tone | 12 giá trị là enum `agent_status`; event status là phép chiếu; 12 → 5 attention → 8 tone là hàm thuần trong `packages/contracts` | ADR-0017 / D-017 | Không |
| CF-010 | `run_status` vs `node_run_status` | DB `run_status` 7 giá trị dùng chung cho `workflow_runs`/`node_runs`; A07: 9 trạng thái node_run, 8 trạng thái run; thiếu `READY/QUEUED/SKIPPED/WAITING_HUMAN`; ORCHESTRATION M-O4: `NodeRun` unique có `revision` nhưng DB không | Thêm enum `node_run_status` và cột `revision` bằng migration; `QUEUED` chỉ là `queuedReason` ở task | ADR-0003 | Có |
| CF-011 | Case enum `task_status` | DECISION_LOG mục 5, DATABASE_DESIGN §12, WORKFLOW_ENGINE, ORCHESTRATION M-O8: tài liệu viết HOA, enum khác viết thường | Mã trạng thái UPPER_SNAKE trong DB và API, nhãn hiển thị qua i18n (ROADMAP §3.14 #1) | ADR-0017, ADR-0003 | Có |
| CF-012 | `provider_kind` thiếu giá trị | DB enum: `claude_code, anthropic_api, openai, gemini, ollama, openrouter, custom` — thiếu `codex`, `openhands`, `generic_cli` | Migration thêm ba giá trị; `provider.kind` ∈ {claude_code, codex, openhands, generic_cli} (BACKEND §3) | ADR-0005, ADR-0006 | Không |
| CF-013 | Cổng approval Claude | CLAUDE_CODE_INTEGRATION và D-004: `canUseTool`; SECURITY §5.1: PreToolUse hook; PERMISSION_MODEL §14 mục 4; PROVIDER §25 #13: `PermissionRequest` hook có thể trùng | Hai tầng dùng chung một policy engine: `canUseTool` là đường chính, PreToolUse là lớp bắt buộc; cập nhật SECURITY §5.1 | ADR-0004 / D-004 | Có |
| CF-014 | Chế độ spawn Claude | WORKSPACE_AND_GIT_ISOLATION: `acceptEdits` + `.claude/settings.json` + hook `command`; SECURITY: `manual` + `--settings` + hook `http`; PROVIDER_INTEGRATION (bảng permissionMode): `default` + whitelist | Mặc định `default` + whitelist an toàn; `acceptEdits`/`auto` chỉ theo role; settings truyền qua `--settings`, không ghi file trong worktree; không `bypassPermissions` ngoài sandbox | ADR-0004, ADR-0009 | Có |
| CF-015 | Approval policy Codex | WORKSPACE/SECURITY: `never` (ví dụ `-c approval_policy="never"`); PROVIDER_INTEGRATION (ví dụ `thread/start`): `on-request` | app-server dùng `on-request` + `sandbox: workspace-write`; `never` chỉ cho fallback `exec` trong Tier 2; không dùng `untrusted` (đã retire) | ADR-0005 | Có |
| CF-016 | Container warm vs ephemeral | SECURITY: 1 container sống lâu per project (`aiw-<projectId>`, `docker exec`); INFRASTRUCTURE_DESIGN §6.1: ephemeral per job | Warm per project cho dev/test, ephemeral cho preview/custom; recreate sau install, 4 giờ hoặc đổi trust (SANDBOX_DESIGN §14 mục 3 cần ADR) | ADR-0007 / D-007 | Có |
| CF-017 | POL-11/POL-12 vs rule R030/R040/R050 | USER_ROLES_AND_PERMISSIONS: POL-11 merge và POL-12 dependency là "tùy chọn" (POL-12 mặc định tắt); PERMISSION_MODEL R030/R040/R050: bắt buộc ask hoặc allow + audit | Merge vào `integration/*` tự động sau Lớp 3 + review L3; vào `main` luôn ask; `dependency_add` ask ở project untrusted, allow + audit ở trusted; Lead không approve thay human | ADR-0009, ADR-0007 | Có |
| CF-018 | Engine Office | SCREEN_SPEC §3.3: "PixiJS/Canvas 2D [Giả định]"; PIXEL_AGENTS_ANALYSIS §8.1: dùng `renderer.ts` Canvas; D-011 và LICENSE_AND_REUSE §5.1: Phaser | Phaser 3.90 sau `OfficeRenderer`; chỉ port logic Pixel Agents; sửa SCREEN_SPEC | ADR-0011 / D-011 | Không |
| CF-019 | Kích thước tile và map | DB `tile_size` 32 và SCREEN "32px tiles" vs nguồn tái dùng 16 px; map mặc định DB 40×24 vs thiết kế 64×40 | Tile logic 32 px, art 16 px phóng nguyên 2×; map mặc định 64×40 (đủ 14 phòng); sửa default DB | ADR-0011 | Có |
| CF-020 | Số animation | DB `character_positions.animation` 7 giá trị (có `sleeping`) vs FR-158 13 giá trị (không `sleeping`) | Theo FR-158 (13); migration enum; `sleeping` chỉ giữ nếu owner yêu cầu | ADR-0011 | Có |
| CF-021 | Màu blocked | SCREEN_SPEC §3.3: bubble blocked/failed đỏ; DESIGN_TOKENS: `blocked` orange, `failed` red; agent-state `busy` violet vs status `running` blue | blocked = orange, failed/error = red; `busy` (vòng avatar) tím, chip trạng thái dùng token status | ADR-0015, ADR-0017 | Không |
| CF-022 | Relax/Coffee khi idle | SCREEN §3.3: idle không đi uống cà phê, không họp giả lập; FR-151 (14 phòng cố định) và FR-164 (Relax/Coffee khi Idle) | Phòng luôn có; nhân vật chỉ vào theo event + policy, mặc định tắt (luật không giả lập tiến độ) | ADR-0011, ADR-0017 | Có |
| CF-023 | Agent `waiting_approval` đứng đâu | SCREEN §3.3: đi tới Review desk vs ít chuyển động giả | Mặc định đứng tại desk; tùy chọn `goToReviewDesk` | ADR-0011 | Không |
| CF-024 | OpenHands ACP vs D-008 | PROVIDER_INTEGRATION §9.5: ACP materialize `CODEX_AUTH_JSON` vào `~/.codex/auth.json` trong container, trái D-008 (app không đọc/copy `auth.json`) | MVP chỉ OpenHands native + LLM key trong OS keychain; ACP tắt cho tới khi owner duyệt | ADR-0006, ADR-0008 | Có |
| CF-025 | Giấy phép sprite JIK-A-4 | PIXEL_AGENTS_ANALYSIS §0/§5.3: chưa xác minh; LICENSE_AND_REUSE §1.3/§6.1: CC0 đã xác minh (web search, không lưu bằng chứng); VIRTUAL_OFFICE §21 #2 | Coi là chưa xác minh tới khi chụp trang license itch.io vào PR; lấy sprite từ nguồn CC0 gốc, không qua bản Pixel Agents | ADR-0012 / D-012 | Không |
| CF-026 | CC0-only vs D-012 | Yêu cầu hiện tại "asset chỉ CC0" vs D-012 và LICENSE_AND_REUSE §6.2: cho phép MIT/OFL/CC-BY có file license (VIRTUAL_OFFICE §21 #1) | Cho phép CC0/MIT/OFL kèm THIRD_PARTY_NOTICES; CC-BY chỉ khi owner duyệt từng gói | ADR-0012 | Có |
| CF-027 | Turborepo | CI_CD.md: "pnpm 10 + Turborepo [Cần người dùng phê duyệt]"; D-001 chỉ ghi pnpm workspaces; không nghiên cứu so sánh (TECHNOLOGY_EVALUATION TE-09) | Bắt đầu pnpm thuần; thêm Turborepo khi > ~5 package hoặc CI vượt ngưỡng; sửa CI_CD tương ứng | ADR-0001 | Có |
| CF-028 | Test runner | CHATDEV_ANALYSIS §11.1: port test sang Jest; NestJS mặc định Jest; frontend/đề bài: Vitest (TE-11, ARCHITECTURE_COMPARISON C-5) | Vitest cho `apps/web` và `packages/*`; `apps/api` dùng Vitest + `unplugin-swc` nếu spike qua, nếu không thì Jest chỉ cho api; một runner mỗi package | ADR-0001 | Có |
| CF-029 | Số loại node workflow | FR-113: 8 loại; yêu cầu hiện tại: 9 (có `Task`); DB comment: `agent_task`/`join`; A07: `agent`/`merge` | 11 `type` theo WORKFLOW_ENGINE §3.2, thêm kind `task` | ADR-0003 | Có |
| CF-030 | Template mặc định: stage và auto-deploy | A07: 8 stage, gate 1,2,5,6,7, deploy production; FR-119..127: 8 stage khác, gate 2,4,8, không auto-deploy production | Theo FR-119..127; không auto-deploy production | ADR-0003, ADR-0007 | Có |
| CF-031 | PM chấm điểm | FR-109: PM agent chấm điểm; A07 §6 và A13: scoring xác định, PM chỉ can thiệp khi bế tắc | Scoring xác định bằng mã, PM agent chỉ xử lý bế tắc; sửa FR-109 | ADR-0003, ADR-0016 | Có |
| CF-032 | Tên branch/worktree | A07: `wt/<run>/<node>/<attempt>` và `wf/<runId>/<nodeId>/r<rev>`; D-009 và WORKSPACE: `agent/<slug>/<task-id>` | `agent/<slug>/<task-id>` cho branch; đường dẫn worktree theo `<task-id>` | ADR-0009 / D-009 | Không |
| CF-033 | Vị trí DECISION_LOG | Đề bài tham chiếu `docs/DECISION_LOG.md`; file thực tế `AI-Workspace-App/DECISION_LOG.md` (ARCHITECTURE_COMPARISON C-7) | Giữ ở thư mục gốc; sửa tham chiếu; README ADR trỏ về file gốc | README ADR | Không |
| CF-034 | Đường dẫn SECURITY.md và DATABASE_DESIGN.md | File thực tế `docs/SECURITY.md` và `docs/DATABASE_DESIGN.md`; INFRASTRUCTURE_DESIGN, WORKSPACE_AND_GIT_ISOLATION, CI_CD trích `docs/security/SECURITY.md` và `docs/architecture/DATABASE_DESIGN.md` | Chuẩn hóa về `docs/SECURITY.md` và `docs/DATABASE_DESIGN.md`; chủ sở hữu từng tài liệu sửa tham chiếu | ADR-0007, ADR-0002 | Không |
| CF-035 | Thiếu bảng `event_outbox` | API_DOCUMENTATION §13 liệt kê chặn freeze; DATABASE_DESIGN 44 bảng không có outbox; A07 có `EventOutbox` | Quyết định theo CF-004: nếu dùng con trỏ thì thêm bảng `event_relay_cursor`; nếu giữ outbox thì thêm `event_outbox`; chốt trước P1-04 | ADR-0014 | Có |
| CF-036 | Thiếu bảng template version và các bảng phụ | AGENT_ARCHITECTURE §3.1.1: DB chỉ có `agents.is_template`, thiếu `template_version`, skills kèm trọng số; thiếu `notification_reads` (HERDR_DESIGN §2) và cột `node_runs.revision` | Thêm `agent_templates` + `agent_template_versions`, `notification_reads`, cột `revision` bằng migration; instance pin `templateVersion` | ADR-0016, ADR-0017 | Có |
| CF-037 | Thiếu AC cho nhóm FR MVP | IMPLEMENTATION_ROADMAP §7 mục 11: không có AC cho FR-001..011 Company/Department, Dashboards FR-181..195, Approval UI, Premium UI; ACCEPTANCE_CRITERIA chỉ 100 AC, có khoảng trống số (AC-016..020, 035..040, 065..070) | BA bổ sung AC cho bốn nhóm trước khi thoát Phase 1; ghi khoảng trống số là dự trữ | IMPLEMENTATION_ROADMAP | Không |
| CF-038 | Số lần retry tối đa | A13 `maxRetries=3`, A07 `maxAttempts=3`, DB `agents.max_retry`/`tasks.max_retry` = 2 (AGENT_ARCHITECTURE M-08) | Giá trị lấy từ `tasks.max_retry`, mặc định cấu hình chung 3; owner chốt 2 hay 3 | ADR-0003, ADR-0016 | Có |
| CF-039 | Fallback giữa provider | DB `agent_provider_connections.priority` (fallback) vs A13 "không auto-failover"; RESOURCE: không auto-switch provider | Không auto-failover; `priority` chỉ gợi ý cho owner chuyển thủ công | ADR-0010, ADR-0004 | Có |
| CF-040 | Tên interface adapter | CLAUDE_CODE_INTEGRATION §9.1: `BrainAdapter`/`BrainEvent` (`claude-code`/`codex-cli`); CODEX_INTEGRATION §11.2: `AgentEvent`, `AgentBrain` (`codex`/`claude`) | Hợp nhất `AgentAdapter`/`AgentEvent` trong `@aiw/agent-core` (BACKEND §3) | ADR-0004, ADR-0005 | Không |
| CF-041 | Xác thực | DATABASE_DESIGN §8: JWT/session; SECURITY §7: bearer per-launch | Desktop dùng bearer, server mode dùng JWT (BACKEND §3.1) | ADR-0001 | Không |
| CF-042 | Quy ước đặt tên hạ tầng | `apps/backend` (CODEX) vs `apps/api` (CI_CD, DATABASE); `workspaceId` vs `organization_id`; Redis `ws:workspace:<id>` và `events:<workspaceId>` vs `aiw:ch:*`, `aiw:evt:session:{id}:buffer`; id ULID vs UUID; queue `node.agent.*` vs `agent-runner`/`workflow-engine` (SYSTEM_ARCHITECTURE §8 #4–6, EVENT_SYSTEM M-E1/M-E4, WORKFLOW M-W4) | `apps/api`; `organizationId`; mọi key Redis có tiền tố `aiw:`; UUID v7; tên queue chốt trong P1 | ADR-0001, ADR-0014 | Không |
| CF-043 | CLI trên host vs "không chạy code AI trên host" | DECISION_LOG mâu thuẫn 1, SYSTEM_ARCHITECTURE §6, PROJECT_RISKS: yêu cầu gốc vs Tier 1 | Chấp nhận D-007 hai tầng; owner ký R1, R2, R3, R5, R6 bằng văn bản | ADR-0007 / D-007 | Có |
| CF-044 | 18 agent vs quota và điều khoản dịch vụ | DECISION_LOG mâu thuẫn 2, THREAT_MODEL §7, PROJECT_RISKS R-TOS-QUOTA: ToS nhiều phiên song song chưa xác minh | Owner xác minh điều khoản và gói subscription; hiển thị gauge quota; mặc định 5 | ADR-0010, ADR-0004 | Có |
| CF-045 | Khoảng trống chưa kiểm chứng | DECISION_LOG mâu thuẫn 3 và 4: Docker daemon chưa chạy (cgroup, ECI, rootless), Codex CLI chưa cài (mọi test Codex chưa chạy) | Bật Docker Desktop và cài `@openai/codex` để chạy SB-00..SB-17 và live test Codex trước Phase 4 | ADR-0005, ADR-0007 | Có |
| CF-046 | Schema manifest quyền file | WORKSPACE: JSON, `readOnly`; SECURITY §6.2: YAML, `readonly`; vị trí trong/ngoài worktree | JSON `schemaVersion: 1`, key `readOnly`, thêm `maxLinesChanged`/`forbidden`, đặt ngoài worktree | ADR-0009 | Không |
| CF-047 | Cấu hình/env của Claude | RESOURCE §4.3: `CLAUDE_CONFIG_DIR` riêng (chưa chứng minh giữ OAuth login); research Claude: `settingSources:['user']`; env whitelist SECURITY thiếu `APPDATA/LOCALAPPDATA` mà research yêu cầu | Dùng `settingSources` theo research; chỉ thử `CLAUDE_CONFIG_DIR` sau khi kiểm chứng login còn; thêm `APPDATA/LOCALAPPDATA` vào whitelist | ADR-0004, ADR-0008 | Có |
| CF-048 | `WAITING_APPROVAL` có chiếm slot | RESOURCE §4.1: không chiếm slot; PROVIDER §17: approval mức tool giữ process sống nên chiếm slot | Approval mức tool chiếm slot (có ngưỡng thời gian rồi trả về `WAITING_HUMAN`); approval mức task không chiếm | ADR-0003, ADR-0010 | Không |
| CF-049 | API đọc quota | RESOURCE §8.1: chưa có API đọc quota Claude; §8.2: Codex hoàn toàn giả định; PROVIDER §25: `rate_limit_event.unifiedWindows` đã xác minh và `account/rateLimits/read` có trong bindings | Sửa RESOURCE §8 theo PROVIDER; dùng hai nguồn đó cho gauge | ADR-0010, ADR-0005 | Không |
| CF-050 | Tên event approval Codex | SECURITY §5.1: `exec_approval_request` (v1 legacy); app-server v2: `item/commandExecution/requestApproval` | Dùng tên v2; cập nhật SECURITY | ADR-0005 | Không |
| CF-051 | Hash chain audit và phạm vi | DEVSECOPS §9: hash chain tùy chọn sau MVP; SECURITY §8: bắt buộc; DATABASE_DESIGN: agent action vào `agent_events`; SECURITY §8: `tool_call/tool_result` vào audit | Bật hash chain từ M0; tách `agent_events` (hoạt động) và `audit_logs` (quyết định/cưỡng chế), đồng bộ `correlation_id` | ADR-0008, ADR-0014 | Có |
| CF-052 | Bảng `secrets` vs `credential_ref` | CREDENTIAL_MANAGEMENT §14: DATABASE_DESIGN không có bảng `secrets`; chỉ `credential_ref` | Chỉ `credential_ref` (khóa keychain), không bảng `secrets` ở MVP | ADR-0008 / D-008 | Có |
| CF-053 | Worktree: host-truth vs volume-truth | SANDBOX_DESIGN §14 mục 2 chưa chốt; ảnh hưởng §4.5 | Host-truth (worktree trên host, bind vào container chỉ phần `writable`) cho MVP | ADR-0009, ADR-0007 | Có |
| CF-054 | Phạm vi tái dùng UI OpenHands | LICENSE_AND_REUSE §3.1: ALLOW UI component, hooks; OPENHANDS_ANALYSIS §11.3: UI gắn chặt HeroUI/i18n/PostHog/Cloud (ARCHITECTURE_COMPARISON C-6) | UI là reference-only; chỉ tái dùng contract/client/ingress/process utils | ADR-0006, ADR-0012 | Không |
| CF-055 | Vai trò duyệt workflow vs RBAC | WORKFLOW_ENGINE M-W5: `approverRoles` (`product`, `tech_lead`, `qa_lead`) vs DB `org_role`: owner/admin/member/viewer | Bảng ánh xạ vai trò agent/người sang quyền duyệt, đối chiếu USER_ROLES_AND_PERMISSIONS trước P3 | ADR-0003, ADR-0016 | Có |

Tổng: 55 mâu thuẫn (Có = 37, Không = 18 theo cột cuối).

---

## Cập nhật 2026-10-30 sau phản hồi của Owner
Nguồn: `PHASE_0_OWNER_DECISION_REGISTER.md` mục 8 và `docs/DOC_SYNC_LOG_PHASE0.md` mục "Đồng bộ theo quyết định Owner đã duyệt".
- **Đã đóng bằng đồng bộ tài liệu theo quyết định đã duyệt:** CF-001 (D-002), CF-002 (D-010), CF-016, CF-043, CF-053 phần host (D-007), CF-027, CF-028, CF-041, CF-042 phần `apps/api` (D-001), CF-049.
- **Vẫn hoãn:** CF-004, CF-035, CF-010, CF-036 (liên quan D-002 nhưng còn phụ thuộc D-014 và Q-12), CF-038 (số lần retry: 2 hay 3, chưa chốt), CF-048 (ngưỡng thời gian approval), phần đặt tên Redis, ID và queue của CF-042, phần D-009 của CF-053.
- **Mới phát sinh:** Q-16 (khung frontend Next.js hay Vite), Q-17 (D-004), Q-18 (D-021 và bốn FR), Q-19 (event naming trong DB), Q-20 (D-003).
- Kiểm tra độc lập sau đồng bộ: không còn đường dẫn `prisma/**` trong cấu hình; seed `max_concurrent_agents` là 5; không có thay đổi nào về Vite hay Next.js ngoài một dòng đánh dấu chờ xác nhận (Q-16).


---

## Cập nhật OWR-2 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md` (OWR-2), đối chiếu `PHASE_0_OWNER_DECISION_REGISTER.md` mục 9 đến 11. Các đoạn "Quy tắc bắt buộc: chủ sở hữu chưa duyệt bất kỳ quyết định nào" và cột `PENDING_OWNER_APPROVAL` ở trên là nội dung lịch sử lúc lập (phiên bản 2); **trạng thái hiện hành là bảng dưới đây**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

Trạng thái dùng ở mục này: `RESOLVED_BY_OWNER_DECISION` = Owner đã quyết định (cần kèm Q/D); **việc đồng bộ tài liệu chưa được xác nhận (doc sync pending)**, do agent khác thực hiện; `STILL_OPEN` = OWR-2 không giải quyết hoặc còn điều kiện, kèm lý do. Không dòng nào dưới đây là VERIFIED.

### A. Đóng bằng quyết định của Owner (doc sync pending)

| CF | Quyết định | Nguồn OWR-2 | Kết quả | Việc đồng bộ còn lại |
|---|---|---|---|---|
| CF-003 | Tên event: DB dùng dotted lowercase (`task.started`, `agent.blocked`); API/lớp tương thích có thể ánh xạ UPPER_SNAKE; không hai hệ thống độc lập (phương án A) | Q-12, Q-19 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | EVENT_SYSTEM.md M-E2 (dòng 39,72,172,407), VIRTUAL_OFFICE.md; bảng ánh xạ 1-1 trong `@aiw/contracts` |
| CF-004 | Transactional Outbox, một relay leader, Redis Pub/Sub hoặc Streams, replay `lastEventId` (phương án A: `event_outbox`) | D-014 | RESOLVED_BY_OWNER_DECISION (doc sync pending). CF-035 (thiếu bảng `event_outbox`) cùng hướng, doc sync pending | Sửa ADR-0014 nội dung gốc "đọc theo `global_seq`" thành đọc `event_outbox`; DATABASE_DESIGN thêm bảng. Danh sách event gốc xem CF-056 (mới) |
| CF-006 | Audit online 365 ngày; signed export 2 năm | Q-06 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | SECURITY.md:498 `180 ngày` -> 365 ngày + export 2 năm |
| CF-007 | Approval wait tối đa 540 s, rồi lưu trạng thái, chuyển `WAITING_HUMAN`, không giữ worker; nhắc 24 giờ; deploy production/xóa dữ liệu/merge main duyệt riêng, hết hạn 15 phút | Q-05 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | Sáu giá trị còn lệch trong SECURITY.md:400, BACKEND_ARCHITECTURE.md:149, DATABASE_DESIGN.md:832 (3600 s), PROVIDER_INTEGRATION.md:411 (10 phút), RESOURCE_MANAGEMENT.md:104, WORKFLOW_ENGINE.md:201,359, research Claude (30 phút) |
| CF-048 | Cùng Q-05: tool-level chờ tới 540 s rồi `WAITING_HUMAN` không chiếm worker | Q-05 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | RESOURCE_MANAGEMENT.md:55,104 theo bản sửa của CF-048 |
| CF-008 | L0-L5 ánh xạ `permission_level` + `permission_profile`; không `full` | Q-03 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | DATABASE_DESIGN.md:170 bỏ `full`, thêm CHECK; chốt số level (AGENT_ORGANIZATION.md:30 ghi L0..L4) |
| CF-011 | `task_status` UPPERCASE; enum khác lowercase | Q-13 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | DATABASE_DESIGN.md §12 ghi rõ quy ước |
| CF-013 | Claude Agent SDK là adapter chính; raw CLI chỉ hỗ trợ/kiểm tra/PoC | D-004, Q-17 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | SECURITY.md §5.1 viết lại theo PROVIDER_INTEGRATION §17 |
| CF-014 | Cùng D-004 (SDK chính). OWR-2 **không** nêu chế độ spawn `default` + whitelist cụ thể | D-004, Q-17 | RESOLVED_BY_OWNER_DECISION cho hướng SDK; chế độ permission mặc định là chi tiết thiết kế, **chưa được Owner nêu**, coi là STILL_OPEN nhỏ | WORKSPACE_AND_GIT_ISOLATION.md:116,129; SECURITY.md:94,114 |
| CF-015, CF-050 | Hoãn Codex khỏi MVP | D-005 | RESOLVED_BY_OWNER_DECISION ở nghĩa hoãn phạm vi; mở lại khi Codex quay lại | WORKSPACE_AND_GIT_ISOLATION.md:132 ghi chú hoãn |
| CF-017 | Agent merge `integration/*` khi qua gate; `main` luôn cần Owner; Lead không thay Owner | Q-04, D-009 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | USER_ROLES_AND_PERMISSIONS.md:119,120,148,159 POL-11/12; PERMISSION_MODEL R030/R040/R050. Quy tắc `dependency_add` ở project tin cậy/chưa tin cậy **không được OWR-2 nêu**, giữ như đề xuất cũ chưa duyệt |
| CF-018 | Phaser 3.90 qua `OfficeRenderer` | D-011 | RESOLVED_BY_OWNER_DECISION, kèm điều kiện benchmark trước khi khóa (doc sync pending) | SCREEN_SPECIFICATION.md:142,690 |
| CF-009 | 5 attention state (`blocked`, `working`, `done`, `idle`, `unknown`) chỉ suy ra từ lifecycle event và heartbeat thật | D-017 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | Bảng ánh xạ 12 -> 5 -> 8 tone trong `packages/contracts` |
| CF-036 | 18 role là seed data; tách Agent/Provider/User Identity | D-016, D-017 | RESOLVED_BY_OWNER_DECISION cho hướng seed data; bảng template version là thiết kế còn lại | DATABASE_DESIGN.md thêm bảng template/version |
| CF-021 | Accent tím và token theo D-015 (màu `blocked` cam hay đỏ OWR-2 **không** nêu) | D-015 | STILL_OPEN: OWR-2 chỉ nêu accent, font, density, avatar, locale; chưa nêu màu `blocked` | SCREEN_SPECIFICATION.md:143 so với DESIGN_TOKENS.md:58,183 |
| CF-022 | Idle không tự đi coffee/relax mặc định; có setting tùy chọn | Q-14 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | SCREEN_SPECIFICATION.md:143,678; VIRTUAL_OFFICE.md:157-158 |
| CF-024 | Không tích hợp OpenHands ACP trong MVP | Q-02 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | PROVIDER_INTEGRATION.md:263 (§9.5 #1) |
| CF-026 | Chỉ CC0 mặc định; font OFL kèm license/NOTICE; CC-BY duyệt từng package. MIT cho asset **không** được nêu | Q-15, D-012 | RESOLVED_BY_OWNER_DECISION với chính sách CC0 mặc định; điểm MIT cho asset là PENDING_OWNER_CLARIFICATION (register P-9) | D-012/ADR-0012, LICENSE_AND_REUSE.md:134, VIRTUAL_OFFICE.md §21 |
| CF-029 | 9 UI node types, 11 engine kinds; cập nhật FR-113 | Q-11 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | FR-113, WORKFLOW_ENGINE.md M-W2, SCREEN_SPECIFICATION.md:471 |
| CF-031 | FR-109 dùng công thức deterministic; PM Agent chỉ can thiệp khi mắc kẹt | Q-09 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | FR-109 (FUNCTIONAL_REQUIREMENTS.md:129); công thức cụ thể chưa được Owner nêu |
| CF-039 | Không tự động failover provider khi hết quota; thông báo Owner và chờ | Q-08 | RESOLVED_BY_OWNER_DECISION (doc sync pending) | DATABASE_DESIGN.md:383,391 ghi `priority` chỉ là gợi ý |
| CF-052 | Không plaintext token trong DB hoặc frontend; secret khác dùng OS keychain | D-008 | RESOLVED_BY_OWNER_DECISION (doc sync pending); hướng khớp `credential_ref` | Bảng `secrets` không đưa vào MVP |
| CF-054 | OpenHands hoãn sau Phase 4; tái dùng code MIT/Apache sau xác minh license; asset AGPL cấm | D-006, D-012 | RESOLVED_BY_OWNER_DECISION cho phạm vi (UI OpenHands không dùng trong MVP) (doc sync pending) | LICENSE_AND_REUSE.md, ADR-0006/0012 |

### B. Vẫn mở (STILL_OPEN)

| CF | Lý do còn mở | Nguồn OWR-2 |
|---|---|---|
| CF-019 | Kích thước tile 16 hay 32 không được quyết định; chờ benchmark `OfficeRenderer` và Owner (register P-7) | D-011 |
| CF-020, CF-023 | OWR-2 không nêu 13 animation hay chỗ đứng của `waiting_approval`. D-011 đã APPROVED_CONDITIONAL nhưng điều kiện benchmark chưa đạt; chờ xác nhận có áp dụng đồng bộ ngay không | D-011 |
| CF-025 | Giấy phép sprite JIK-A-4 vẫn chưa có bằng chứng lưu; Q-15 chỉ cho phép CC0 mặc định, nên cần ảnh chụp license trước khi bundle | Q-15 |
| CF-043 | Q-01 chỉ là xác nhận có điều kiện (OWR-1), Owner chưa ký từng ô checklist; thêm security gates OWR-2 §8 | OWR-1 8.1, 8.3; OWR-2 §8 |
| CF-044 | Q-07 chỉ giữ **mặc định 5 của ứng dụng**, KHÔNG xác nhận subscription cho phép 5 session song song. Xác minh ToS/usage limit Claude Team và ChatGPT/Codex vẫn mở (PENDING_OWNER_CLARIFICATION); không tự tăng 10 hoặc 18; không tuyên bố 18 Agent đã test | Q-07, §5.1, §8 |
| CF-045 | Q-10: Docker cần cho sandbox validation, Codex CLI chưa cài trong giai đoạn này. Docker do Owner tự mở thủ công theo §5.3; chưa có kết quả `docker version` / `docker info` mới; SB-00..SB-17 chưa chạy; live test Codex không thực hiện (D-005 hoãn) | Q-10, §5.2, §5.3, §5.6 |
| CF-010 | run_status so với node_run_status: OWR-2 chỉ quyết định số node type (Q-11) và PoC Conductor (Q-20); enum trạng thái run/node chờ PoC | Q-11, Q-20 |
| CF-014 (phần permission mode) | Chế độ permission mặc định của Claude chưa được Owner nêu | D-004 |
| CF-021 (màu `blocked`) | Xem bảng A | D-015 |
| CF-032, CF-046, CF-053 (phần D-009) | OWR-2 D-009 chỉ nêu một worktree/branch mỗi task, review diff, không auto-merge `main`; không nêu tên branch cụ thể, schema manifest, hay worktree host-truth và volume-truth | D-009 |
| CF-038 | Số lần retry tối đa (2 hay 3): OWR-2 chỉ nêu "retry an toàn" trong tiêu chí PoC Q-20, không nêu số | Q-20 |
| CF-042 | Quy ước đặt tên hạ tầng (UUID, tên Redis, queue) không được OWR-2 nêu ngoài tên event (Q-19) | Q-19 |
| CF-051 | Hash chain audit và phạm vi: OWR-2 chỉ nêu retention (Q-06), không nêu phạm vi hash chain | Q-06 |

### C. Xung đột mới phát sinh từ OWR-2 (chưa có mã CF chính thức; mã CF-056.. là tạm)

| Mã tạm | Nội dung | Trạng thái |
|---|---|---|
| CF-056 | Danh sách event "Prompt 1 mục 8.4" không có trong repo (D-014); Owner cần xác nhận bản sao lưu, không tự tạo danh sách | STILL_OPEN (PENDING_OWNER_CLARIFICATION) |
| CF-057 | D-005 hoãn Codex khỏi MVP làm đổi nhóm chấp nhận MVP B của đặc tả gốc (FR-077 M/MVP, AC-022, GP-1, PRD.md:88) | STILL_OPEN: Owner cần biết (PENDING_OWNER_CLARIFICATION) |
| CF-058 | D-015: locale mặc định Vietnamese so với tài liệu thiết kế ghi tiếng Anh mặc định (ADR-0015 nội dung gốc, DESIGN_TOKENS.md, UI_UX_GUIDELINES.md, FRONTEND_ARCHITECTURE.md, SYSTEM_ARCHITECTURE.md) | RESOLVED_BY_OWNER_DECISION (D-015, doc sync pending) |
| CF-059 | Q-15 so với D-012/LICENSE_AND_REUSE: MIT/CC-BY cho asset | STILL_OPEN: MIT cho asset chưa được Owner nêu |
| CF-060 | D-006 hoãn OpenHands, trong khi Tier 2 của ADR-0007 và nhiều tài liệu (PRD.md:88, SRS.md:28,53) coi OpenHands là adapter | STILL_OPEN: Tier 2 chưa có runtime trong MVP, Owner cần biết |
| CF-061 | OWR-2 §8: Phase 2 chỉ build adapter ở mock/sandbox mode nếu security gate chưa đạt, so với IMPLEMENTATION_ROADMAP.md và ADR-0004; chưa đối chiếu chi tiết | STILL_OPEN |
| CF-062 | FR-122, FR-181, FR-182, FR-183: OWR-2 §4 chỉ là đề xuất Acceptance Criteria, chưa APPROVED; phải đối chiếu FR gốc trước khi cập nhật | STILL_OPEN (PENDING_OWNER_CLARIFICATION) |

Ghi chú: các CF còn lại (không liệt kê ở trên) không bị OWR-2 tác động; trạng thái của chúng theo "Cập nhật 2026-10-30 sau phản hồi của Owner" ở trên. Việc đóng CF ở mục này nghĩa là Owner đã quyết định, **không** có nghĩa tài liệu đã được sửa hay đã được kiểm chứng.
