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
