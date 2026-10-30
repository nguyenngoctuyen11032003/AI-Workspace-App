# DOC_SYNC_LOG_PHASE0 — Nhật ký đồng bộ tài liệu Phase 0 (có ràng buộc)

Ngày: 2026-10-30. Phạm vi: chỉ các chỉnh sửa không phụ thuộc quyết định đang chờ (D-001..D-021, Q-xx). Owner chưa duyệt quyết định nào; không mục nào dưới đây được coi là APPROVED. Đường dẫn tính từ `AI-Workspace-App/`. Mọi chỉnh sửa do "Documentation agent" thực hiện. Xác minh = `grep` sau khi sửa (cụm cũ không còn, cụm mới có mặt).

## 1. Đã áp dụng

| Change ID | file:dòng | Cũ (rút gọn) | Mới (rút gọn) | Lý do | Nguồn | Thực hiện | Xác minh |
|---|---|---|---|---|---|---|---|
| DS-01 | docs/architecture/INFRASTRUCTURE_DESIGN.md:5 | `docs/security/SECURITY.md`, `docs/architecture/DATABASE_DESIGN.md` | `docs/SECURITY.md`, `docs/DATABASE_DESIGN.md` | Đường dẫn chéo hỏng | CF-034, QA P2-03 | Documentation agent | `grep -rn "docs/security/SECURITY.md\|docs/architecture/DATABASE_DESIGN.md" docs` = 0 kết quả ngoài adr/QA |
| DS-02 | docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md:5 | như DS-01 | như DS-01 | như DS-01 | CF-034, QA P2-03 | Documentation agent | cùng lệnh DS-01 |
| DS-03 | docs/development/CI_CD.md:5 | `docs/security/SECURITY.md` | `docs/SECURITY.md` | như DS-01 | CF-034, QA P2-03 | Documentation agent | cùng lệnh DS-01 |
| DS-04 | docs/development/DEPLOYMENT.md:5 | như DS-01 | như DS-01 | như DS-01 | CF-034, QA P2-03 | Documentation agent | cùng lệnh DS-01 |
| DS-05 | docs/development/ENVIRONMENT_SETUP.md:5 | `docs/security/SECURITY.md` | `docs/SECURITY.md` | như DS-01 | CF-034, QA P2-03 | Documentation agent | cùng lệnh DS-01 |
| DS-06 | docs/operations/DEVSECOPS_GUIDE.md:5 | như DS-01 | như DS-01 | như DS-01 | CF-034, QA P2-03 | Documentation agent | cùng lệnh DS-01 |
| DS-07 | docs/security/SANDBOX_DESIGN.md:306 | `docs/security/VERIFICATION_LOG.md` (file do Phase 1 tạo...) | `docs/security/SANDBOX_VERIFICATION_LOG.md` [file chưa tạo; sẽ tạo ở Phase 1...] | File không tồn tại | CF-034, QA P2-03 | Documentation agent | `grep -rn "docs/security/VERIFICATION_LOG" docs` = 0 |
| DS-08 | docs/security/SANDBOX_DESIGN.md:339 | `ghi vào VERIFICATION_LOG` | `ghi vào SANDBOX_VERIFICATION_LOG [file chưa tạo]` | như DS-07 | CF-034, QA P2-03 | Documentation agent | `grep -n SANDBOX_VERIFICATION docs/security/SANDBOX_DESIGN.md` = dòng 306, 339 |
| DS-09 | docs/SECURITY.md:35 | `617 bytes, mode 644` | `file người dùng đọc được bởi tiến trình cùng user, kiểm soát bằng ACL` | Bỏ số byte/mode không đúng trên Windows | QA P2-05 | Documentation agent | `grep -rnE "617 ?byte\|mode 644" docs` = 0 ngoài adr/QA |
| DS-10 | docs/SECURITY.md:533 | `dạng file mode 644**` | `dạng file (đọc được bởi tiến trình cùng user, kiểm soát bằng ACL)**` | như DS-09 | QA P2-05 | Documentation agent | cùng lệnh DS-09 |
| DS-11 | docs/research/CLAUDE_CODE_INTEGRATION.md:158 | `(617 bytes)` | `(file người dùng đọc được ... ACL)` | như DS-09 | QA P2-05 | Documentation agent | cùng lệnh DS-09 |
| DS-12 | docs/security/THREAT_MODEL.md:24 | `(mode 644)` | `(file người dùng đọc được ... ACL)` | như DS-09 | QA P2-05 | Documentation agent | cùng lệnh DS-09 |
| DS-13 | docs/security/CREDENTIAL_MANAGEMENT.md:20 | `617 byte, mode 644` | `file người dùng đọc được ... ACL` | như DS-09 | QA P2-05 | Documentation agent | cùng lệnh DS-09 |
| DS-14 | docs/security/CREDENTIAL_MANAGEMENT.md:224 | `OAuth file 644 đọc được` | `OAuth file đọc được` | Cùng chủ đề (bỏ mode), cùng file đã sửa | QA P2-05 | Documentation agent | cùng lệnh DS-09 |
| DS-15 | docs/PROJECT_RISKS.md:68,118 | `file mode 644` | `file ... đọc được bởi tiến trình cùng user (ACL)` | Cùng tuyên bố mode 644 (file ngoài danh sách QA, sửa cho nhất quán) | QA P2-05 | Documentation agent | cùng lệnh DS-09 |
| DS-16 | docs/BACKEND_ARCHITECTURE.md:114 | `**[Đã xác minh]** Danh sách queue` | `**[Nhất quán với DATABASE_DESIGN.md §10 và MULTI_AGENT_ORCHESTRATION.md §2]**` | Đối chiếu nội bộ, không phải bằng chứng lệnh | QA P2-06 | Documentation agent | `grep -n "Nhất quán với" docs/BACKEND_ARCHITECTURE.md` = dòng 114 |
| DS-17 | docs/development/DEVELOPMENT_GUIDE.md:98 | `WORK_PLAN đầu trang) [Đã xác minh]` | `[Nhất quán với WORK_PLAN.md]` (nhãn thứ hai về Codex chưa cài giữ nguyên: có `where codex`) | như DS-16 | QA P2-06 | Documentation agent | `grep -c "Nhất quán với" docs/development/DEVELOPMENT_GUIDE.md` = 3 |
| DS-18 | docs/development/DEVELOPMENT_GUIDE.md:145 | `Orchestrator [Đã xác minh]` | `Orchestrator [Nhất quán với WORK_PLAN.md]` | như DS-16 | QA P2-06 | Documentation agent | cùng lệnh DS-17 |
| DS-19 | docs/development/DEVELOPMENT_GUIDE.md:236 | `\| [Đã xác minh] \|` | `\| [Nhất quán với WORK_PLAN.md] \|` | như DS-16 | QA P2-06 | Documentation agent | cùng lệnh DS-17 |
| DS-20 | docs/architecture/INFRASTRUCTURE_DESIGN.md:151 | `memory=12GB, processors=6 trên máy 32 GB` | giá trị theo RAM máy; máy đo được 15.77 GB nên đề xuất memory khoảng 6–8GB; 12GB/6 chỉ cho máy 32 GB (vẫn `[Cần người dùng phê duyệt]`) | Khuyến nghị giả định máy 32 GB | QA P1-12 | Documentation agent | `grep -c "15.77" docs/architecture/INFRASTRUCTURE_DESIGN.md` = 1 |
| DS-21 | docs/DATABASE_DESIGN.md:29 | `Drizzle (v1.x)` | `Drizzle (v1.x [cần xác minh phiên bản trước khi pin: npm latest 0.45.x, 1.0 đang beta])` | Sửa nhãn phiên bản, không đổi lựa chọn ORM | QA P1-08 (phần nhãn) | Documentation agent | `grep -c "cần xác minh phiên bản" docs/DATABASE_DESIGN.md` = 1 |
| DS-22 | docs/development/ENVIRONMENT_SETUP.md:178 | `(Prisma/Drizzle/TypeORM)` | `(ORM theo D-002, đề xuất, chờ duyệt)` | Diễn đạt trung lập, không chọn ORM | QA P2-02 | Documentation agent | `grep -c "ORM theo D-002" docs/development/ENVIRONMENT_SETUP.md` = 1 |
| DS-23 | docs/design/SCREEN_SPECIFICATION.md:320 | `đã đạt 18 agent → ... "Giới hạn 18 agent đồng thời"` | `đã đạt số agent tối đa theo cấu hình max_concurrent_agents → ... "Giới hạn {max_concurrent_agents} agent đồng thời"` | Chuỗi UI cứng | QA P2-07 | Documentation agent | `grep -c "Giới hạn {max_concurrent_agents}" docs/design/SCREEN_SPECIFICATION.md` = 1 |
| DS-24 | docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md:651 (chèn trước) | (không có) | `[LỖI THỜI] Template chính thức theo WORKFLOW_ENGINE.md §10 (FR-119..127); không auto-deploy production.` | Template A07 lỗi thời; FR do owner đặt | CF-030 (TECH_EVIDENCE) | Documentation agent | `grep -n "LỖI THỜI" docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md` = dòng 651 |
| DS-25 | docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md:~736 (node s8_release) | `"label": "S8 · Release Production"` | `... [minh họa, bị cấm bởi FR-126]` | Đánh nhãn target production | CF-030 | Documentation agent | `grep -c "minh họa, bị cấm bởi FR-126" ...` = 1 |
| DS-26 | docs/research/CLAUDE_CODE_INTEGRATION.md:537 (chèn dưới §9.1) | (không có) | Ghi chú BrainAdapter/BrainEvent là tên nghiên cứu; chính thức AgentAdapter/AgentEvent | Tên interface | CF-040 (NGAY) | Documentation agent | `grep -n "Ghi chú: BrainAdapter" ...` = 1 |
| DS-27 | docs/research/CODEX_INTEGRATION.md:561 (chèn dưới §11.2) | (không có) | Ghi chú AgentBrain là tên nghiên cứu; chính thức AgentAdapter/AgentEvent | như DS-26 | CF-040 (NGAY) | Documentation agent | `grep -n "Ghi chú: AgentBrain" ...` = 1 |
| DS-28 | docs/architecture/RESOURCE_MANAGEMENT.md:111, 115, 219 | không nêu nguồn đọc quota | thêm nguồn `rate_limit_event.unifiedWindows`, `account/rateLimits/read` [Tài liệu, chưa chạy live]; câu hỏi 2 ghi "đã có đường đọc" | Chỉ ghi nguồn tài liệu, giữ nhãn giả định | CF-049 (NGAY) | Documentation agent | `grep -c "Tài liệu, chưa chạy live" ...` = 2; dòng 219 có "(đã có đường đọc" |
| DS-29 | docs/research/LICENSE_AND_REUSE.md:352 | `**CC0 1.0** (hiển thị trên trang)` | `**CC0 1.0 [chưa lưu bằng chứng: cần ảnh chụp trang itch.io trong PR]**` | Nhãn phải phản ánh mức bằng chứng thật | CF-025 (NGAY) | Documentation agent | `grep -c "chưa lưu bằng chứng" ...` = 1 |

## 2. Kiểm tra, không cần sửa

| Item | Kết quả |
|---|---|
| DECISION_LOG D-007 thiếu cột "Phê duyệt" (QA P2-04) | Đã có (`Chờ duyệt, cần xác nhận rủi ro R1–R6`); không sửa |
| CF-033 vị trí DECISION_LOG | ALREADY_FIXED theo CONFLICTS; không còn liên kết `docs/DECISION_LOG.md` (chỉ ARCHITECTURE_COMPARISON C-7 mô tả lịch sử) |
| CF-012 provider_kind | ALREADY_FIXED |
| WORK_PLAN.md "Prisma/Drizzle", "vitest/jest" | Không tìm thấy trong `AI-Workspace-App/` (chỉ còn trong văn bản mô tả mâu thuẫn); không sửa |

## 3. Deferred (blocked by owner decision)

| Mục | file:dòng | Quyết định chặn |
|---|---|---|
| CF-001 prisma/** -> Drizzle path, skill prisma, đổi `packages/db` | AGENT_ORGANIZATION:41,42,66,67,138; PERMISSION_MODEL:101,120; SECURITY:241,376,430,432; DEVELOPMENT_GUIDE:61,63,107; ORCHESTRATION_WORKFLOW_RESEARCH:4 | D-002 |
| CF-002 / P1-04 concurrency mặc định | DATABASE_DESIGN:324,1318; CLAUDE_CODE_INTEGRATION:724; WORKSPACE_AND_GIT_ISOLATION:78,199 | D-010 |
| CF-003 / P1-13 tên event | EVENT_SYSTEM, SYSTEM_ARCHITECTURE, DATABASE_DESIGN §5.2 | D-014, Q (FR-117) |
| CF-004, CF-035 outbox, thêm bảng `event_outbox` (45 bảng) | ADR-0014, DATABASE_DESIGN §4.14 | D-014 |
| CF-005 / P1-01 `audit_events` -> `audit_logs` (NGAY theo CONFLICTS nhưng thuộc nhóm tên bảng audit bị loại khỏi lượt này) | SECURITY:484; INFRASTRUCTURE_DESIGN:185,212; DEPLOYMENT:125,143,163,182; DEVSECOPS_GUIDE:116,174,207; SKILL_SYSTEM:497,645 | Chờ chỉ đạo (audit table name) |
| CF-006 / P1-02 retention audit | SECURITY:498 | Q (D-002) |
| CF-007 / P1-03 timeout approval | SECURITY:400; RESOURCE_MANAGEMENT:104; DATABASE_DESIGN:832 | D-003, D-004 |
| CF-008 / P1-14 mức quyền | DATABASE_DESIGN:170; PERMISSION_MODEL | D-016, D-007 |
| CF-009 agent_status 12/5/8 | BACKEND_ARCHITECTURE:205; SYSTEM_ARCHITECTURE:306; COMPONENT_SPECIFICATION:334; SCREEN_SPECIFICATION:19 | D-017 |
| CF-010 node_run_status (TECH_EVIDENCE) | DATABASE_DESIGN:194-196 | D-003 |
| CF-013, CF-014 Claude approval/spawn mode | SECURITY:94,114,119,135,351-354,551,565; WORKSPACE_AND_GIT_ISOLATION:116,129 | D-004 |
| CF-015, CF-050 Codex approval | WORKSPACE_AND_GIT_ISOLATION:132; SECURITY:207,356 | D-005 |
| CF-016 container warm/ephemeral | INFRASTRUCTURE_DESIGN:45,94; SANDBOX_DESIGN:28 | D-007 |
| CF-017 POL-11/12 | USER_ROLES_AND_PERMISSIONS:119,120,148,159 | D-009 (Q) |
| CF-018 / P2-01 engine Office | SCREEN_SPECIFICATION:142,690 | D-011 |
| CF-019 / P1-07 tile size (TECH_EVIDENCE) | DATABASE_DESIGN:1023-1025; SCREEN_SPECIFICATION:127 | D-011 |
| CF-020 số animation (TECH_EVIDENCE) | DATABASE_DESIGN:1079 | D-011 |
| CF-021, CF-023 màu blocked, waiting_approval | SCREEN_SPECIFICATION:143 | D-015, D-011 |
| CF-027, CF-028 Turborepo, test runner / P2-02 phần vitest-jest | CI_CD:11-151; CHATDEV_ANALYSIS:571,648; AGENT_ORGANIZATION:41 | D-001 |
| CF-030 phần còn lại | (không còn) | không |
| CF-032 tên branch | ORCHESTRATION_WORKFLOW_RESEARCH:88,701 | D-009 |
| CF-036 bảng template/notification_reads | DATABASE_DESIGN | D-016, D-017 |
| CF-037 / P0-1 viết AC còn thiếu | ACCEPTANCE_CRITERIA, TESTING:11 | Việc viết nội dung (BA), không phải sửa cơ học |
| CF-038 retry | AGENT_SCHEDULER:50,186; AGENT_ARCHITECTURE:281 | D-010 |
| CF-041, CF-042 xác thực, UUID/đặt tên hạ tầng | DATABASE_DESIGN:1267; EVENT_SYSTEM:397,406; CODEX_INTEGRATION:620,648,686; ORCHESTRATION_WORKFLOW_RESEARCH:219,238,578 | D-001, D-014 |
| CF-046 manifest quyền file | SECURITY:428-432 | D-009 |
| CF-047 CLAUDE_CONFIG_DIR (TECH_EVIDENCE) | RESOURCE_MANAGEMENT:70; DEVSECOPS_GUIDE:141 | D-004, D-008 |
| CF-048 WAITING_APPROVAL chiếm slot | RESOURCE_MANAGEMENT:55,104 | D-010 |
| CF-051, CF-052 hash chain, bảng secrets | DEVSECOPS_GUIDE:123,174; SECURITY:297,488; CREDENTIAL_MANAGEMENT:60,210 | D-008 |
| CF-053 worktree host-truth (TECH_EVIDENCE) | SANDBOX_DESIGN:16,48,49,93,145 | D-007, D-009 |
| CF-054 tái dùng UI OpenHands | LICENSE_AND_REUSE:122,205 | D-012, D-006 |
| CF-055 approverRoles | WORKFLOW_ENGINE:95,426 | D-003 |
| P1-05 SECURITY §5 viết lại | SECURITY:119,135,351-354 | D-004 |
| P1-09 tên biến mock, `'mock'` trong ProviderKind | CI_CD:152,168,265; DEPLOYMENT:159; TESTING:94; PROVIDER_INTEGRATION:35 | Q (mock env var), D-005, D-006 |
| P1-08 phần ORM/spike | DATABASE_DESIGN, TECHNOLOGY_EVALUATION | D-002 |
| P2-08 deploy production wording | DEPLOYMENT §9 vs WORKFLOW_ENGINE V12 | Chưa kiểm; thuộc mặc định workflow/policy, chờ chỉ đạo |
| P2-09 Phaser 3.90 vs 4.x | D-011 | D-011 |
| CF-029, CF-031, CF-022, CF-024, CF-026, CF-039, CF-043..045 | nhiều file | OWNER_DECISION (Q-xx) |

## 4. Số lượng

- Đã áp dụng: 29 change ID (trong 22 file), gồm 1 mục TECH_EVIDENCE (CF-030, 2 chỉnh sửa), 3 mục NGAY ngoài danh sách chỉ định (CF-040, CF-049, CF-025; không phụ thuộc quyết định nào).
- Đã đúng sẵn, không sửa: 4 (D-007 Phê duyệt, CF-033, CF-012, WORK_PLAN).
- Deferred: 37 dòng ở bảng mục 3 (đã nhóm theo CF/QA).
- TECH_EVIDENCE_RESOLVABLE: 7; áp dụng 1 (CF-030), hoãn 6 (CF-004, 010, 019, 020, 047, 053), tất cả hoãn do điều kiện `SAU D-xxx`.

## Bổ sung sau QA lần hai (F-02)
| Change ID | File | Thay đổi | Lý do | Nguồn | Người thực hiện | Kiểm tra |
|---|---|---|---|---|---|---|
| SYNC-QA2-01 | SECURITY.md:484, architecture/INFRASTRUCTURE_DESIGN.md:185,212, development/DEPLOYMENT.md:125,143,163,182, operations/DEVSECOPS_GUIDE.md (dòng 5, 106, 116, 142, 147, 174, 207) | `audit_events` đổi thành `audit_logs` | Là đổi tên bí danh cũ, `security/AUDIT_LOGGING.md` mục 4 đã xác định tên bảng chuẩn. Không phụ thuộc quyết định chờ duyệt (không chọn ORM, retention hay schema cột). Danh sách cột khác nhau giữa SECURITY §8 và `audit_logs` vẫn còn, đã ghi chú tại chỗ và để Architect chốt cột | CF-005, QA lần hai F-02 | Orchestrator | `grep -rn audit_events docs` ngoài adr, QA, log, AUDIT_LOGGING cho 0 dòng |
| SYNC-QA2-02 | architecture/RESOURCE_MANAGEMENT.md:19,184,185; research/MULTI_AGENT_SCALABILITY.md:19,27,153,156; BACKEND_ARCHITECTURE.md:60,101,171,173,176; development/DEVELOPMENT_GUIDE.md:202 | Nhãn `[Đã xác minh]` đổi thành `[Tài liệu tự báo cáo, chưa tái hiện]` (số đo) hoặc `[Nhất quán với tài liệu nguồn]` (đối chiếu nội bộ) | Số đo do một agent đo một lần, chưa ai tái hiện (xem PHASE_0_TECHNICAL_VALIDATION_REPORT.md) | QA lần hai F-04 | Orchestrator | `grep -c "Tài liệu tự báo cáo"` > 0 |


## Đồng bộ theo quyết định Owner đã duyệt (2026-10-30)

Nguồn: `PHASE_0_OWNER_DECISION_REGISTER.md` mục 8 (D-001 trừ khung frontend, D-002, D-007, D-010 APPROVED). Không có quyết định nào khác được đánh dấu APPROVED. Không chạm: ADR-*.md, CONFLICTS_TO_RESOLVE.md, DECISION_LOG.md, register, DYNAMIC_SKILL_INJECTION.md, IMPLEMENTATION_ROADMAP.md, traceability_parts. Không thay đổi câu nào về Vite so với Next.js (Q-16 còn mở); chỉ thêm marker một lần ở `FRONTEND_ARCHITECTURE.md`. Q-07 (điều khoản dịch vụ) vẫn mở.

### Bảng thay đổi đã áp dụng

| Change ID | file:dòng (trước khi sửa) | Cũ (rút gọn) | Mới (rút gọn) | Quyết định / CF | Xác minh |
|---|---|---|---|---|---|
| OS-01 | docs/architecture/AGENT_ORGANIZATION.md:41 | nestjs, typescript, prisma, jest | nestjs, typescript, drizzle, vitest | D-002/CF-001; D-001/CF-028 | grep: mới có, cũ không còn |
| OS-02 | docs/architecture/AGENT_ORGANIZATION.md:42 | `prisma/**`, `docker/**`, migrations | sql, prisma, docker-compose | `apps/api/src/database/**`, `drizzle/**`, `docker/**`, migrations | sql, drizzle, docker-c... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-03 | docs/architecture/AGENT_ORGANIZATION.md:66 | `contracts/**`, `prisma/**` | | `contracts/**`, `apps/api/src/database/**`, `drizzle/**` | | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-04 | docs/architecture/AGENT_ORGANIZATION.md:67 | | db_infra | `prisma/**`, `docker/**`, `infra/local/**` | `apps/api/**... | | db_infra | `apps/api/src/database/**`, `drizzle/**`, `docker/**`, `infra/local/**` | `ap... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-05 | docs/architecture/AGENT_ORGANIZATION.md:138 | code: 'prisma' | code: 'drizzle' | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-06 | docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md:4 | **Platform:** Windows 11  | **Platform:** Windows 11  > Prisma được thay bằng Drizzle theo D-002 (APPROVED). Mã Prisma... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-07 | docs/development/DEVELOPMENT_GUIDE.md:262 | 6. Chọn Drizzle hay Prisma: WORK_PLAN §2 còn ghi "Prisma/Drizzle", DEC... | 6. ORM: Drizzle (stable 0.45.x) theo D-002 (APPROVED, Owner 2026-10-30). Tác vụ bắt buộc t... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-08 | docs/MULTI_AGENT_ORCHESTRATION.md:325 | | Drizzle thắng; mã giả Prisma trong A07 chỉ là minh họa | | | Drizzle thắng (D-002 APPROVED, 2026-10-30); mã giả Prisma trong A07 chỉ là minh họa | | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-09 | docs/WORKFLOW_ENGINE.md:431 | | M-W10 | A07 dùng Prisma; D-002 Drizzle | Drizzle | | | M-W10 | A07 dùng Prisma; D-002 Drizzle | Drizzle (D-002 APPROVED, 2026-10-30) | | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-10 | docs/SYSTEM_ARCHITECTURE.md:300 | | Theo D-002; coi Prisma schema trong ORCHESTRATION chỉ là minh họa | | | Theo D-002 (APPROVED, 2026-10-30); coi Prisma schema trong ORCHESTRATION chỉ là minh họa... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-11 | docs/SYSTEM_ARCHITECTURE.md:103 | Cần cập nhật ownership glob trong seed của `AGENT_ORGANIZATION.md` khi... | Ownership glob trong `AGENT_ORGANIZATION.md` đã đổi sang `apps/api/src/database/**`, `driz... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-12 | docs/DATABASE_DESIGN.md:29 | Drizzle (v1.x [cần xác minh phiên bản trước khi pin: npm latest 0.45.x... | Drizzle (stable 0.45.x; 1.0 còn beta, không dùng nhãn "v1.x") | D-002 | grep: mới có, cũ không còn |
| OS-13 | docs/DATABASE_DESIGN.md:1468 | 1. **ORM = Drizzle** thay vì Prisma (lý do mục 1). | 1. **ORM = Drizzle** (stable 0.45.x) thay vì Prisma (lý do mục 1); D-002 APPROVED (Owner, ... | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-14 | docs/research/TECHNOLOGY_EVALUATION.md:22 | | D-002 | Cần phê duyệt | | | D-002 | APPROVED (Owner, 2026-10-30) | | D-002 | grep: mới có, cũ không còn |
| OS-15 | docs/SECURITY.md:241 | `prisma migrate (deploy|reset)` | `drizzle-kit (migrate|push|drop)` | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-16 | docs/SECURITY.md:376 | `src/core`, `schema.prisma`, | `src/core`, `apps/api/src/database/schema/**`, | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-17 | docs/SECURITY.md:430 | "prisma/migrations/2026*_billing*/**" | "drizzle/2026*_billing*/**" | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-18 | docs/SECURITY.md:432 | "prisma/schema.prisma"] | "apps/api/src/database/schema/**"] | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-19 | docs/security/PERMISSION_MODEL.md:101 | "prisma/schema.prisma"] | "apps/api/src/database/schema/**"] | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-20 | docs/security/PERMISSION_MODEL.md:120 | "prisma migrate"] | "drizzle-kit migrate"] | D-002/CF-001 | grep: mới có, cũ không còn |
| OS-21 | docs/DATABASE_DESIGN.md:324 | maxConcurrent: integer('max_concurrent').notNull().default(2), | maxConcurrent: integer('max_concurrent').notNull().default(5), // D-010: mặc định 5; seed ... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-22 | docs/DATABASE_DESIGN.md:1318 | (`max_concurrent_agents = 3`, `max_cost_usd_per_day = 20`). | (`max_concurrent_agents = 5` theo D-010, trần cấu hình 18; `max_cost_usd_per_day = 20`). S... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-23 | docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md:16 | semaphore global 10 + per-provider trong Redis (Lua script). | semaphore global 10 + per-provider trong Redis (Lua script). [Số 10 đã bị thay thế bởi D-0... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-24 | docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md:428 | Per-provider cap đề xuất: `claude: 6`, `codex: 6`, global `10` (cấu hì... | Per-provider cap đề xuất: `claude: 6`, `codex: 6`, global `10` (cấu hình ở `workspace_sett... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-25 | docs/research/CLAUDE_CODE_INTEGRATION.md:726 | Semaphore `MAX_PARALLEL_AGENTS=10`; đọc | Semaphore `MAX_PARALLEL_AGENTS=10` [số nghiên cứu lịch sử; mặc định khuyến nghị: D-010 APP... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-26 | docs/research/CLAUDE_CODE_INTEGRATION.md:486 | `MAX_PARALLEL_AGENTS` (mặc định 10) + semaphore. | `MAX_PARALLEL_AGENTS` (mặc định 10 trong nghiên cứu; thay bằng mặc định 5, trần 18 theo D-... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-27 | docs/research/CODEX_INTEGRATION.md:531 | (mặc định 2–3 turn song song/account) | (mặc định 2–3 turn song song/account trong nghiên cứu; giá trị áp dụng: cap provider Codex... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-28 | docs/AGENT_ARCHITECTURE.md:280 | Cấu hình `ResourceConfig` thắng; DB default chỉ là giá trị khởi tạo [Đ... | Cấu hình `ResourceConfig` thắng (D-010 APPROVED: Claude 5, Codex 3); DB default `max_concu... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-29 | docs/MULTI_AGENT_ORCHESTRATION.md:322 | cap provider do `ResourceConfig`; cần owner chốt giá trị provider | | cap provider do `ResourceConfig`, giá trị theo D-010 APPROVED: Claude 5, Codex 3; DB defau... | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-30 | docs/architecture/RESOURCE_MANAGEMENT.md:49 | (default: claude-code 5, codex 3 [Giả định]) | (default: claude-code 5, codex 3; D-010 APPROVED 2026-10-30) | D-010/CF-002 | grep: mới có, cũ không còn |
| OS-31 | docs/security/SANDBOX_DESIGN.md:28 | luôn ephemeral. [Đề xuất, Cần người dùng phê duyệt] | luôn ephemeral. [D-007 APPROVED (Owner, 2026-10-30); đóng CF-016. Điều kiện: fail-closed k... | D-007/CF-016 | grep: mới có, cũ không còn |
| OS-32 | docs/security/SANDBOX_DESIGN.md:271 | | Đề xuất; cần hoà giải INFRA §6.1 | | | D-007 APPROVED (2026-10-30); đã hoà giải INFRA §6.1 | | D-007/CF-016 | grep: mới có, cũ không còn |
| OS-33 | docs/security/SANDBOX_DESIGN.md:329 | 2. Worktree host-truth vs volume-truth chưa chốt (Runtime team); ảnh h... | 2. ĐÃ CHỐT (D-007/CF-053): host-truth, worktree nằm trên host, chỉ bind phần `writable` và... | D-007/CF-053 | grep: mới có, cũ không còn |
| OS-34 | docs/security/SANDBOX_DESIGN.md:330 | 3. Hợp nhất mô hình container warm (SECURITY) vs ephemeral (INFRA) cần... | 3. ĐÃ CHỐT (D-007/CF-016): warm per project cho dev/build/test, ephemeral cho preview/cust... | D-007/CF-016 | grep: mới có, cũ không còn |
| OS-35 | docs/architecture/INFRASTRUCTURE_DESIGN.md:96 | không bao giờ reuse container giữa các task. | không bao giờ reuse container giữa các task. [Ngoại lệ theo D-007/CF-016 (APPROVED): conta... | D-007/CF-016 | grep: mới có, cũ không còn |
| OS-36 | docs/SYSTEM_ARCHITECTURE.md:279 | | D-007 hai tầng; cần Owner xác nhận R1–R6 (`SECURITY.md` §10) | | | D-007 hai tầng (APPROVED có điều kiện, Owner 2026-10-30; CF-043): fail-closed khi Docker... | D-007/CF-043 | grep: mới có, cũ không còn |
| OS-37 | docs/development/CI_CD.md:11 | (monorepo pnpm + Turborepo) | (monorepo pnpm workspaces, chưa dùng Turborepo; D-001 APPROVED) | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-38 | docs/development/CI_CD.md:20 | pnpm 10 + Turborepo [Cần người dùng phê duyệt]. | pnpm 10 workspaces thuần, chưa dùng Turborepo (D-001 APPROVED, 2026-10-30; thêm Turborepo ... | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-39 | docs/development/CI_CD.md:255 | [Cần người dùng phê duyệt] (cùng quyết định tool monorepo) | | Hoãn: D-001 APPROVED chọn pnpm thuần, chưa dùng Turborepo | | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-40 | docs/development/CI_CD.md:281 | 1. pnpm/Turborepo vs npm/Nx – cần phê duyệt trước khi viết `turbo.json... | 1. ĐÃ CHỐT (D-001): pnpm workspaces thuần, không có `turbo.json` ở Phase 1. | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-41 | docs/development/CI_CD.md:263 | thống nhất tên trong `turbo.json`. | thống nhất tên giữa các `package.json` (không dùng `turbo.json`, D-001). | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-42 | docs/SYSTEM_ARCHITECTURE.md:75 | pnpm 10 + Turborepo, đang là "Cần người dùng phê duyệt") | pnpm 10 workspaces thuần; D-001 APPROVED, chưa dùng Turborepo) | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-43 | docs/SYSTEM_ARCHITECTURE.md:80 | ├─ turbo.json                     # [Cần người dùng phê duyệt] Turbore... | ├─ (không turbo.json)             # D-001 APPROVED: pnpm thuần, Turborepo hoãn | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-44 | docs/SYSTEM_ARCHITECTURE.md:267 | | S-7 | Turborepo + pnpm 10 | Cache build trong monorepo | [Cần người ... | | S-7 | pnpm 10 workspaces thuần (Turborepo hoãn) | Ít công cụ ở Phase 1 | D-001 APPROVED ... | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-45 | docs/SYSTEM_ARCHITECTURE.md:289 | 1. Turborepo có dùng không (CI_CD L20 đang chờ duyệt)? | 1. ĐÃ CHỐT (D-001): không dùng Turborepo ở Phase 1. | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-46 | docs/development/DEVELOPMENT_GUIDE.md:30 | pnpm-workspace.yaml  turbo.json  .nvmrc | pnpm-workspace.yaml  .nvmrc | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-47 | docs/development/DEVELOPMENT_GUIDE.md:133 | | `turbo.json`, `tsconfig.base.json`, | | `tsconfig.base.json`, | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-48 | docs/development/DEVELOPMENT_GUIDE.md:230 | Monorepo pnpm + Turborepo, ranh giới | Monorepo pnpm workspaces (không Turborepo), ranh giới | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-49 | docs/development/DEVELOPMENT_GUIDE.md:230 | [Đề xuất] theo D-001 (chờ duyệt) | D-001 APPROVED (Owner, 2026-10-30) | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-50 | docs/development/ENVIRONMENT_SETUP.md:185 | turbo/pnpm -r chạy song song | pnpm -r chạy song song | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-51 | docs/development/ENVIRONMENT_SETUP.md:270 | | Turborepo làm task runner | Cache lint/test/build theo package, hỗ t... | | Turborepo làm task runner | Cache lint/test/build theo package, hỗ trợ `--filter` và rem... | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-52 | docs/development/ENVIRONMENT_SETUP.md:290 | 1. pnpm vs npm; Turborepo vs Nx – cần phê duyệt. | 1. ĐÃ CHỐT (D-001): pnpm workspaces thuần, không Turborepo. | D-001/CF-027 | grep: mới có, cũ không còn |
| OS-53 | docs/research/CHATDEV_ANALYSIS.md:571 | port thành Jest tests | port thành Vitest tests (CF-028, D-001 APPROVED; Jest chỉ cho `apps/api` nếu spike `unplug... | D-001/CF-028 | grep: mới có, cũ không còn |
| OS-54 | docs/research/CHATDEV_ANALYSIS.md:648 | Port sang TS/Jest | Port sang TS/Vitest | D-001/CF-028 | grep: mới có, cũ không còn |
| OS-55 | docs/research/CODEX_INTEGRATION.md:622 | // apps/backend/src/agents/codex/codex-app-server.client.ts | // apps/api/src/agents/codex/codex-app-server.client.ts | D-001/CF-042 | grep: mới có, cũ không còn |
| OS-56 | docs/research/CODEX_INTEGRATION.md:650 | // apps/backend/src/agents/codex/codex.adapter.ts | // apps/api/src/agents/codex/codex.adapter.ts | D-001/CF-042 | grep: mới có, cũ không còn |
| OS-57 | docs/research/CODEX_INTEGRATION.md:688 | // apps/backend/src/agents/codex/codex-exec.runner.ts | // apps/api/src/agents/codex/codex-exec.runner.ts | D-001/CF-042 | grep: mới có, cũ không còn |
| OS-58 | docs/DATABASE_DESIGN.md:1267 | `TenantContext` (AsyncLocalStorage) set từ JWT/session; | `TenantContext` (AsyncLocalStorage) set từ JWT/session ở server mode (desktop mode dùng be... | D-001/CF-041 | grep: mới có, cũ không còn |
| OS-59 | docs/FRONTEND_ARCHITECTURE.md:22 | Vite 7 + TypeScript 5.9 · React 19.2 | **Khung frontend: chờ Owner xác nhận (Q-16)** · Vite 7 + TypeScript 5.9 · React 19.2 | Q-16 (marker, không đổi nội dung) | grep: mới có, cũ không còn |

Tổng: 59 thay đổi trong 20 file. Không chạy git add/commit/push; thay đổi để chưa commit.

### CF đã đóng bởi đợt đồng bộ này (CONFLICTS_TO_RESOLVE.md do tiến trình khác sở hữu, chưa sửa ở đây)

| CF | Đóng theo | Ghi chú |
|---|---|---|
| CF-001 | D-002 | Prisma bỏ khỏi mọi lựa chọn ORM (OS-01..OS-11, OS-15..OS-20). Bảng so sánh Prisma vs Drizzle trong DATABASE_DESIGN, TECHNOLOGY_EVALUATION, ARCHITECTURE_COMPARISON và mã Prisma trong ORCHESTRATION_WORKFLOW_RESEARCH giữ nguyên làm lịch sử. Spike partition và FK vòng là tác vụ bắt buộc trước khi khóa schema, CHƯA thực hiện. |
| CF-002 | D-010 | Mặc định 5, trần 18, Claude 5, Codex 3 (OS-21..OS-30). Số nghiên cứu 10/6 giữ làm lịch sử kèm ghi chú. |
| CF-016 | D-007 | Warm per project cho dev/build/test, ephemeral cho preview/custom (OS-31, OS-32, OS-34, OS-35). |
| CF-027 | D-001 | pnpm thuần, chưa Turborepo (OS-37..OS-52). Các ví dụ `pnpm turbo` trong CI_CD giữ làm minh họa kèm ghi chú. |
| CF-028 | D-001 | Vitest cho web và packages; `apps/api` Vitest + unplugin-swc nếu spike qua, nếu không Jest (OS-01, OS-53, OS-54). TESTING.md đã dùng Vitest từ trước. |
| CF-041 | D-001 | Desktop bearer per-launch, server mode JWT (OS-58). |
| CF-042 | D-001 (chỉ phần `apps/api`) | OS-55..OS-57. Các phần organizationId, tiền tố Redis `aiw:`, UUID v7, tên queue phụ thuộc D-014, xem bảng "Vẫn hoãn". |
| CF-043 | D-007 | OS-36. Duyệt có điều kiện: fail-closed khi Docker tắt; SB-00..SB-17 qua trên máy có Docker trước khi mở quyền thực thi; Owner chấp nhận rủi ro ghi overlay ở Phase 0-1 (checksum + audit). Việc ký từng ô R1, R2, R3, R5, R6 vẫn do Owner tự thực hiện (Q-01). |
| CF-049 | D-010 | Không cần sửa thêm: đã áp dụng ở DS-28 (nguồn `rate_limit_event.unifiedWindows`, `account/rateLimits/read` trong RESOURCE_MANAGEMENT §8). |
| CF-053 | D-007 | Host-truth (OS-33). Phần ADR-0009 (D-009) vẫn chờ duyệt. |

Đã thêm marker `Khung frontend: chờ Owner xác nhận (Q-16)` (OS-59). Web-first và Tauri 2 sau đã khớp sẵn trong DEPLOYMENT.md, SYSTEM_ARCHITECTURE.md, nên không cần sửa.

### Vẫn hoãn

| CF / hạng mục | Vì sao chưa áp dụng | Quyết định chặn |
|---|---|---|
| CF-004 outbox vs event store | Hàng CF ghi phụ thuộc D-014; register D-002 chỉ ghi "liên quan" | D-014 (chờ duyệt, Q-12, Q-19) |
| CF-035 bảng event_outbox / event_relay_cursor | Phụ thuộc kết quả CF-004 | D-014 |
| CF-036 agent_templates, notification_reads | Hàng CF phụ thuộc D-016, D-017 | D-016, D-017 |
| CF-010 node_run_status | Hàng CF phụ thuộc ADR-0003 (workflow engine) | D-003 (Q-20) |
| CF-038 số lần retry | Mơ hồ: hàng CF ghi "mặc định chung 3; owner chốt 2 hay 3", register khuyến nghị `max_retry` 2; Owner chỉ duyệt phần concurrency của D-010 | Owner cần chốt 2 hay 3 (liên quan D-010, ADR-0003) |
| CF-048 WAITING_APPROVAL chiếm slot | Giá trị khuyến nghị gắn ngưỡng thời gian approval và ADR-0003; approval timeout còn chờ | D-003, D-004, Q-05 |
| CF-042 phần organizationId, tiền tố Redis, UUID v7, tên queue | Phụ thuộc quy ước event/hạ tầng | D-014 |
| CF-053 phần ADR-0009 (worktree per task) | D-009 chưa duyệt | D-009 |
| ToS subscription cho nhiều phiên song song | Q-07 còn mở | Q-07 |
| Vite so với Next.js | Owner viết Next.js, đề xuất là React + Vite | Q-16 |
| Thực thi spike Drizzle (partition, FK vòng) | Là việc triển khai Phase 1, không phải đồng bộ tài liệu | Phase 1 |
