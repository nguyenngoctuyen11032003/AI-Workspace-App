# PHASE_0_QA_REVIEW_2 — QA độc lập lần hai, Phase 0 AI WORKSPACE

Ngày: 2026-10-30. Reviewer: QA & Technical Reviewer độc lập (không viết/sửa tài liệu nào được rà). Chỉ tài liệu. Chỉ chạy lệnh đọc (grep, wc, find, node script đọc file, `claude --version/--help`, `docker version`, `herdr --version`, `codex.exe --version`, `npm view`, `git status`). Không cài, không bật Docker, không chạy prompt model, không đọc file credential. Script đối chiếu nằm ở scratchpad (ngoài repo). File duy nhất tạo ra: file này.

## 0. Verdict điều hành

**Verdict chung: PASS WITH FIXES.** Không có P0. Các lỗi P0 của vòng 1 (traceability AC, enum `provider_kind`) đã được xử lý có bằng chứng; số liệu tự khai trong `PHASE_0_FINAL_REPORT.md` đúng gần hết. Điểm yếu còn lại: 4 FR mức Must vẫn chưa có AC, một sửa lỗi cơ học đã xếp NGAY nhưng không áp dụng (`audit_events`), nhãn "Đã xác minh" sai mức bằng chứng còn nằm trong tài liệu nguồn, tài liệu theo dõi có vài câu sai/lỗi thời, và DYNAMIC_SKILL_INJECTION có lỗ hổng thiết kế chống agent tự sửa overlay chưa được vá.

Đếm: **P0 = 0, P1 = 3, P2 = 8, P3 = 6.**

| # | Hạng mục kiểm tra | Verdict |
|---|---|---|
| 1 | Tài liệu bắt buộc tồn tại, đủ nội dung | PASS (file này là deliverable thứ 7; xem §1) |
| 2 | FR coverage và AC integrity | PARTIAL (số liệu khớp 100%; 4 Must FR chưa có AC; số "6 Must/Should" sai, thực tế 9) |
| 3 | 25 mục vòng 1 | PARTIAL (xem §3: 9 fixed, 4 partial, 11 deferred có lý do, 1 open lý do yếu) |
| 4 | Nhất quán, DOC_SYNC_LOG, số liệu FINAL_REPORT | PASS WITH FIXES (không có APPROVED giả; spot-check sync đúng; số liệu đúng trừ 2 chi tiết) |
| 5 | Evidence honesty | PARTIAL (file finalization trung thực; tài liệu nguồn vẫn còn nhãn sai) |
| 6 | DYNAMIC_SKILL_INJECTION.md | PARTIAL (nhất quán, không lén sửa FR/AC/DB; có lỗ hổng bảo mật chưa phủ) |
| 7 | Security signoff sanity | PASS |
| 8 | Tuân thủ quy trình | PASS |

## 1. Tài liệu bắt buộc

Lệnh: vòng `for` với `[ -f ]` và `wc -l` cho 57 file trong bảng §1.2 của vòng 1 và 15 file gốc; `find docs -empty`.
- 57/57 file `docs/**` tồn tại, không file nào < 60 dòng (không in dòng `SHORT`), `find docs -empty` rỗng. (Vòng 1 ghi "63/63"; bảng của vòng 1 liệt kê 57 file, chênh lệch là cách đếm, không thiếu file.)
- Deliverable: FINAL_REPORT 63 dòng, TRACEABILITY 532, OWNER_DECISION_REGISTER 331, TECHNICAL_VALIDATION 302, SECURITY_SIGNOFF 234, REMAINING_RISKS 110; QA_REVIEW_2 = file này.
- Tracking: AGENTS 56, NEXT_SESSION 53, PROJECT_STATUS 34, WORK_PLAN 54, TASK_BACKLOG 60, DECISION_LOG 39, KNOWN_ISSUES 20, IMPLEMENTATION_LOG 18, AGENT_WORK_LOG 51. `docs/adr/` có 20 ADR + README + TEMPLATE + CONFLICTS (vòng 1: rỗng).

## 2. FR coverage và AC integrity (node script, kết quả thật)

| Kiểm tra | Kết quả |
|---|---|
| FR trong FUNCTIONAL_REQUIREMENTS (hàng `\| FR-xxx`) | 199 duy nhất (211 mã xuất hiện, gồm mã chỉ nhắc trong văn bản) |
| Dòng FR trong bảng truy vết | 199, 199 duy nhất; missing = [], extra = [], duplicates = [] |
| Trạng thái | AC_DEFINED 109 + AC_ADDED_PHASE0 74 + NEEDS_OWNER_CLARIFICATION 16 = 199, khớp mọi con số khai báo |
| Priority trong bảng truy vết so với FR file | 0 sai lệch (141 M, 45 S, 13 C) |
| AC được trích (171 mã) mà không định nghĩa ở ACCEPTANCE_CRITERIA hoặc 3 file addendum | 0 |
| Trùng ID | AC gốc 100 duy nhất; không trùng giữa gốc và addendum; AC-211 xuất hiện 2 lần chỉ vì một dòng ghi chú, không phải định nghĩa trùng. Dải 201-219 (19), 301-320 (20), 401-433 (33), 097-100 (4): không va chạm |
| 8 hàng AC_DEFINED (AC-001, 014, 056, 071, 126, 011, 073, 004) | 8/8 tồn tại, đúng FR. Quét tự động 227 cặp (FR, AC): 3 cặp mà AC gốc không liệt kê FR đó (FR-102/AC-122, FR-127/AC-053, FR-197/AC-045), nội dung vẫn hợp lý |
| 18 AC mới lấy mẫu (AC-201, 205, 209, 214, 215, 219, 301, 305, 310, 316, 320, 401, 405, 411, 418, 425, 428, 433) | 15/18 có Given/When/Then đo được (con số, "0 khác biệt", trạng thái cụ thể). Yếu: AC-305 có câu "không kiểm" ngay trong Then; AC-425 gộp nhiều hành vi và một câu hỏi mở (9 so với 8 loại node); AC-411 để ý "IDE xem Q-05" lơ lửng |
| 30 ID AC chưa định nghĩa của vòng 1 | AC-097..100 được định nghĩa trong addendum (không nằm trong ACCEPTANCE_CRITERIA.md); AC-016..020, 035..040, 065..070, 083..085, 115..120 giữ "dự trữ", được ghi rõ. Grep xác nhận AC-017, AC-040 không có hàng định nghĩa |
| 16 NEEDS_OWNER_CLARIFICATION | 12/16 là khoảng trống thật (định nghĩa "sự cố", công thức "tiến độ", cơ chế OpenHands, API key fallback, số cấp quản lý...). Nhưng 4 hàng là **Must** (FR-122, FR-181, FR-182, FR-183) và 5 hàng Should (FR-006, 009, 088, 186, 283): tổng 9 M/S, **không phải "6"** như TRACEABILITY dòng 18. FR-181 liệt kê sẵn chỉ số trong chính FR nên viết được một phần AC |

Phát hiện phụ: TRACEABILITY §1 nói vòng 1 "50 FR thiếu AC" còn đợt này "90", rồi ghi "chưa được đối soát chi tiết": trung thực nhưng chưa giải thích. Mã Q-xx lặp trong từng phần (Q-06 vừa là FR-006 vừa là FR-186); có ghi chú tiền tố P1-/P2-/P3- nhưng dễ nhầm.

## 3. 25 mục vòng 1

| Mục | Trạng thái | Bằng chứng |
|---|---|---|
| P0-1 AC traceability | **Fixed có điều kiện** | 199/199 FR (§2). Chưa hợp nhất vào ACCEPTANCE_CRITERIA.md (chỉ con trỏ dòng 197); `TESTING.md:11` và `IMPLEMENTATION_ROADMAP.md:5` vẫn "AC-001…AC-130" |
| P0-2 provider_kind | **Fixed** | `DATABASE_DESIGN.md:171-175` enum chứa `codex, openhands, generic_cli, mock, ...`. Còn `'mock'` trong enum production (vòng 1 khuyến nghị bỏ); `PROVIDER_INTEGRATION.md:35` là tập 5 giá trị không có kind API |
| P1-01 audit_events | **Open** (lý do yếu) | `grep -rn audit_events docs` ngoài adr/QA/sync-log = 17 dòng. CONFLICTS:118-119 "NGAY" nhưng DOC_SYNC_LOG §3 hoãn "chờ chỉ đạo": không phụ thuộc quyết định nào |
| P1-02 retention 180/365 | Deferred, hợp lý | `SECURITY.md:498` vẫn 180; AUDIT_LOGGING:158 là 365 (Đề xuất) |
| P1-03 approval timeout | Deferred (D-003, D-004) | DOC_SYNC_LOG §3 |
| P1-04 seed concurrency 3 | Deferred (D-010) | `DATABASE_DESIGN.md:1318` còn `max_concurrent_agents = 3` |
| P1-05 SECURITY §5 hook HTTP | Deferred (D-004) | SECURITY:119,135,351-354 |
| P1-06 prisma trong manifest | Deferred (D-002) | `AGENT_ORGANIZATION.md:41,42,66` còn `prisma/**` |
| P1-07 tile 32/16 | Deferred (D-011) | `DATABASE_DESIGN.md:1025` default 32 |
| P1-08 Drizzle v1.x | **Partial** | `DATABASE_DESIGN.md:29` ghi "cần xác minh phiên bản"; `npm view drizzle-orm dist-tags.latest` = 0.45.4 (tôi chạy) |
| P1-09 3 biến mock | Deferred | `CI_CD.md:152`, `TESTING.md:94`, `DEPLOYMENT.md:159` vẫn 3 tên |
| P1-10 ADR rỗng | **Fixed** | 20 ADR; ADR-0012 nêu MVO REF-ONLY clean-room |
| P1-11 tracking files | **Fixed** (có lỗi nhỏ, F-05) | PROJECT_STATUS 34, KNOWN_ISSUES 20, NEXT_SESSION 53 |
| P1-12 .wslconfig 12GB | **Fixed** | `INFRASTRUCTURE_DESIGN.md:151` "memory khoảng 6-8GB, máy đo 15.77 GB" |
| P1-13 event / "Prompt 1 mục 8.4" | Deferred | `API_DOCUMENTATION.md:347,433` vẫn nêu nguồn không tìm thấy |
| P1-14 permission levels | Deferred (D-016, D-007) | DATABASE_DESIGN:170 |
| P2-01 PixiJS | Deferred (D-011) | `SCREEN_SPECIFICATION.md:142,690` |
| P2-02 Prisma/Drizzle, vitest/jest | **Partial** | `WORK_PLAN.md` hết; `ENVIRONMENT_SETUP.md:178` đã đổi; `AGENT_ORGANIZATION.md:41` còn (deferred) |
| P2-03 liên kết hỏng | **Fixed** | grep 4 đường dẫn cũ ngoài adr/QA/sync-log: chỉ còn 3 dòng mô tả lịch sử |
| P2-04 D-007 thiếu cột | **Fixed** | DECISION_LOG D-007 có "Chờ duyệt" |
| P2-05 617 byte / mode 644 | **Partial** | `grep -rnE "617 ?byte\|mode 644"` = 0; còn `SECURITY.md:574` "(644)" (regex DS-09 không bắt) |
| P2-06 nhãn Đã xác minh | **Partial** | Chỉ sửa BACKEND:114, DEV_GUIDE:98,145,236. Còn BACKEND_ARCHITECTURE:60,101,171,173,176 và DEV_GUIDE:202 |
| P2-07 UI cứng 18 | **Fixed** | `grep -c "Giới hạn {max_concurrent_agents}"` = 1 |
| P2-08 deploy production | Deferred | DEPLOYMENT:137 "không bao giờ tự động"; WORKFLOW_ENGINE V12 chưa đối chiếu ("Chưa kiểm" trong sync log) |
| P2-09 Phaser 3.90 | Deferred, ADR có lý do | ADR-0011:46; `npm view phaser dist-tags.latest` = 4.2.1 (tôi chạy) |

Tổng: fixed 9 (P0-1 có điều kiện, P0-2, P1-10, 11, 12, P2-03, 04, 07 và một mục tính ở dưới: P1-08 tính partial), partial 4 (P1-08, P2-02, P2-05, P2-06), deferred có lý do 11, open lý do yếu 1.

## 4. Nhất quán và số liệu

- Grep `APPROVED|Đã duyệt|đã phê duyệt` (loại phủ định, bảng từ vựng, ô chưa tick, tên trường mã nguồn): không câu nào khẳng định owner đã duyệt. `PROJECT_STATUS.md:27-28` ghi "Chưa có". ADR: 20/20 `PENDING_OWNER_APPROVAL`, mức bằng chứng 20/20 khớp README (10 Unverified, 10 Partial, 0 Verified).
- DECISION_LOG có D-001..D-021 (21 hàng). Sổ quyết định có D-001..D-020 trong bảng tổng hợp và D-021 ở mục chi tiết (dòng 323). ADR README chỉ phủ D-001..D-020, chưa có ADR cho D-021.
- DOC_SYNC_LOG: 29 hàng DS-xx. Spot-check bằng grep DS-01, 07, 09, 16, 17, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29: cụm cũ không còn, cụm mới có mặt (ví dụ `Giới hạn {max_concurrent_agents}` = 1, `LỖI THỜI` dòng 651, `Tài liệu, chưa chạy live` = 2, `chưa lưu bằng chứng` = 1). Không thay đổi nào bake giá trị phụ thuộc quyết định chờ duyệt (DS-20 vẫn gắn `[Cần người dùng phê duyệt]`, DS-21/22 trung lập). Lỗi: log ghi "22 file", tôi đếm 20 file khác nhau.
- "Deferred" giấu thay đổi đã áp dụng? mtime cho thấy 5 file sửa sau vòng 1 mà **không có trong log**: `AGENT_ARCHITECTURE.md:297`, `PROVIDER_INTEGRATION.md:528`, `SKILL_SYSTEM.md:700`, `WORKSPACE_AND_GIT_ISOLATION.md:338` (phụ lục D-021, PROPOSED) và `ACCEPTANCE_CRITERIA.md:197` (con trỏ). Nội dung chỉ là con trỏ, vô hại, nhưng log tự nhận đầy đủ.
- Số liệu FINAL_REPORT: 199 FR **đúng**; 109/74/16 **đúng**; 55 mâu thuẫn 15/7/33 **đúng** (đếm 55 hàng: 15 OWNER_DECISION, 7 TECH_EVIDENCE, 31+2 STALE); 29 sync **đúng**; "22 file" trong sync log **sai** (20); 13 NOT_ACCEPTABLE **đúng** (13 ACCEPTABLE_IF_OWNER_ACKNOWLEDGES cũng đúng); 62 rủi ro / 17 chặn **đúng** (62 dòng; 17 dòng "yes"); 27 runbook **đúng** (VAL-CODEX/DOCKER/CONC mỗi loại 9); 20 ADR **đúng**, 10 Unverified/10 Partial **đúng**; "74 AC mới" là 74 FR được gán AC mới, thực tế 76 mã AC mới (72 mới + 4 khôi phục AC-097..100), lệch đơn vị đếm nhỏ; RAM trống "3,9 GB" so với "4,18 GB" ở checklist (thời điểm khác; lúc tôi đo 1,78 GB nên dao động là thật).

## 5. Evidence honesty

Lệnh tôi chạy lại: `claude --version` = `2.1.312 (Claude Code)` (khớp); `docker version` = Client 29.8.1, server lỗi pipe `dockerDesktopLinuxEngine` (khớp, daemon không chạy); `herdr --version` = `herdr 0.9.5` (khớp); `codex.exe --version` bản bundled trong VS Code extension `openai.chatgpt-26.51021.22027-win32-x64` = `codex-cli 0.164.0-alpha.11.1` (khớp); `where codex` = không tìm thấy (khớp); `node -v` v24.19.0, `pnpm -v` 10.34.6, `gh --version` 2.102.0, `git --version` 2.56.0 (khớp); `npm view` drizzle-orm 0.45.4, phaser 4.2.1, @openai/codex 0.166.0 (khớp ADR/Register). `claude --help` liệt kê `--append-system-prompt`, `--add-dir`, `--system-prompt-snapshot <on|off>` (khớp), nhưng **không liệt kê `--append-system-prompt-file` thành tùy chọn riêng**: chỉ có ký hiệu `--append-system-prompt[-file]` trong mô tả `--bare`. DYNAMIC_SKILL_INJECTION 4.5 bước 3 và Register D-021 ghi "liệt kê ... Đã xác minh": mức bằng chứng yếu hơn chữ viết (F-09).

Trung thực của file finalization: TECHNICAL_VALIDATION phân biệt rạch ròi "Verified now" và "Reported_by_agent_not_reproduced"; không file nào nói đã chạy live test (grep "18 agent đã đạt/chứng minh" = không có; RISKS TECH-03 và VALIDATION dòng 4 ghi chưa chứng minh). **Nhưng** tài liệu gốc mà report đó phê phán vẫn giữ nhãn cũ: `RESOURCE_MANAGEMENT.md:19,184,185` "[Đã xác minh]" cho số đo 1/3 process; `MULTI_AGENT_SCALABILITY.md` 6 chỗ; `CLAUDE_CODE_INTEGRATION.md` 1 chỗ. KNOWN_ISSUES #9 tự nhận. Chưa được vá.

## 6. DYNAMIC_SKILL_INJECTION.md

Nhất quán: các mục tham chiếu (SKILL_SYSTEM 4.2/4.3.1/4.3.2/4.4/4.5, D8, dòng 306 "không hot-reload"; WORKSPACE manifest `forbidden` dòng 93 chưa có `.ai/**`; AGENT_ARCHITECTURE 3.5.1/3.9) đều có thật và mô tả đúng. Claim về snapshot (mặc định bật, dùng lại nguyên văn khi resume, tới khi compact, `off` render lại) khớp từng ý với `claude --help` thật. Không sửa lén phạm vi: `grep "FR-293|AC-501|skills_config|\.ai/\*\*"` ngoài chính file này = 0 kết quả; FR-293..298 và AC-501..512 chỉ là đề xuất; DB/FR/AC không bị đổi. 12 điểm S1-S12 đúng như FINAL_REPORT nói.

| Mối đe dọa | Doc phủ? | Nhận xét |
|---|---|---|
| Path traversal / symlink / junction / tên thiết bị Windows | Có (4.3 bước 2, 4.7.4) | Thiếu: NTFS ADS, tên 8.3, TOCTOU giữa `realpath` và đọc |
| Agent tự sửa `.ai/**` | **Phủ một phần, có lỗ hổng** | 4.7.1 dựa vào deny `Write(.ai/**)` và "lớp kiểm diff". (a) 4.1 khuyến nghị đưa `.ai/` vào git dự án: bản checkout trong worktree sửa được, merge về main sẽ đổi overlay (diff gate cần chặn tường minh). (b) Tier 1 chạy trên host không OS sandbox (SECURITY R1): agent ghi đường dẫn tuyệt đối vào `<project_root>/.ai/` bằng Bash/PowerShell không đi qua deny Write; kiểm diff chỉ thấy worktree. (c) Không phân biệt được "Owner sửa" và "agent sửa" (`SKILL_OVERLAY_CHANGED` "người sửa nếu biết") |
| Cách ly giữa agent (4.7.8) | Chỉ mức logic | Trên host Tier 1 mọi agent đọc được thư mục agent khác qua Bash; ghi rõ là policy, không phải boundary |
| Prompt injection | Có (lint heuristic, Owner chịu trách nhiệm) | Chấp nhận được, đã nói rõ không thay kiểm soát quyền |
| Rò bí mật | Có (quét `sk-`, Bearer, khóa, redaction, chặn file) | Heuristic; `system-fragment.md` ghi ra `data/runtime/skills-share/...` đọc được bởi tiến trình cùng user; chưa nêu ACL/dọn dẹp |
| Overlay cấp quyền | Có (loại `allowed-tools`, `permissions`, `hooks`, `mcp`, `env`) | Tốt |
| YAML frontmatter độc hại | Không nêu | Cần parser an toàn và giới hạn |

## 7. Security signoff sanity

`PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md` §2 có 14 hàng: boundary/sandbox (1), filesystem (2), command execution (3), network (4), OAuth/session credentials (5), prompt injection (6), agent-to-agent (7), git worktree isolation (8), audit log (9), human approval (10), emergency stop (11), cộng Docker socket (12), secrets in logs (13), localhost API auth (14): phủ đủ 11 mục owner yêu cầu. Mỗi hàng có design reference, design status, verification (Not tested), residual, ô sign-off để trống. §5: 13 ACCEPTABLE_IF_OWNER_ACKNOWLEDGES (mỗi hàng có điều kiện) và 13 NOT_ACCEPTABLE_UNTIL_MITIGATED (mỗi hàng có mitigation bắt buộc và mốc). Không rủi ro nào thiếu mitigation hoặc quyết định owner. Không ô nào đã tick. DYNAMIC_SKILL_INJECTION chưa được đưa vào checklist (xem F-03).

## 8. Process compliance

- Không có mã ứng dụng: `find . -type f ! -name "*.md"` ngoài `.git` = rỗng; không có thư mục `apps|packages|src|node_modules`, không `package.json`.
- Repo tham chiếu: `git -C <repo> status --porcelain` cho ChatDev, OpenHands, herdr, my-virtual-office, pixel-agents = rỗng (sạch cả 5).
- AI-Workspace-App là một git repo (2 commit, có remote) với 43 mục thay đổi, đều là `.md` hoặc thư mục `docs/adr/`, `docs/product/traceability_parts/`.
- "18 agent đồng thời": không tài liệu nào khẳng định đã đạt (grep). Mọi nơi ghi "mặc định 5, trần 18, chưa chứng minh".

## 9. Bảng phát hiện

| ID | Sev | Phát hiện | Bằng chứng (lệnh → kết quả) | File:dòng | Đề xuất sửa |
|---|---|---|---|---|---|
| F-01 | P1 | 4 FR Must chưa có AC; tài liệu nói "6 FR Must/Should", thực tế 9 (4 M + 5 S) | node script: NEEDS rows M = FR-122, 181, 182, 183; S = 006, 009, 088, 186, 283 | PHASE_0_ACCEPTANCE_CRITERIA_TRACEABILITY.md:18 (FR-181 dòng 177) | Sửa số; viết AC một phần cho FR-181..183 theo chỉ số đã nêu trong FR hoặc ghi lý do hoãn; ưu tiên Owner trả lời Q-01, Q-03..Q-05 |
| F-02 | P1 | audit_events→audit_logs xếp NGAY nhưng không áp dụng, 17 dòng còn lại | `grep -rn audit_events docs` (ngoài adr/QA/log) = 17; CONFLICTS:118-119 "NGAY"; DOC_SYNC_LOG §3 "chờ chỉ đạo" | SECURITY.md:484; INFRASTRUCTURE_DESIGN.md:185,212; DEPLOYMENT.md:125,143,163,182; DEVSECOPS_GUIDE.md:116,174,207; SKILL_SYSTEM.md:497,645 | Áp dụng theo AUDIT_LOGGING:44 và ghi vào DOC_SYNC_LOG |
| F-03 | P1 | Overlay `.ai/**` bị agent sửa được qua Bash trên Tier 1 hoặc qua merge nếu `.ai/` nằm trong repo; diff gate không thấy thư mục ngoài worktree; không có kiểm tra provenance/integrity | Đọc 4.1, 4.7.1, 4.7.8; SECURITY R1; WORKSPACE:93 `forbidden` chưa có `.ai/**` | docs/architecture/DYNAMIC_SKILL_INJECTION.md §4.1, 4.7.1, 4.7.8, 9 | Thêm: ACL ghi chỉ-Owner hoặc overlay ngoài repo có hash đã xác nhận, so hash trước mỗi task, chặn merge đổi `.ai/**`, mount read-only ở Tier 2; đưa vào SECURITY_SIGNOFF |
| F-04 | P2 | Nhãn "[Đã xác minh]" cho số đo chưa tái hiện còn trong tài liệu nguồn dù VALIDATION_REPORT thừa nhận chưa tái hiện | grep RESOURCE_MANAGEMENT:19,184,185; MULTI_AGENT_SCALABILITY (6 chỗ); BACKEND_ARCHITECTURE:60,101,171,173,176 | các file trên | Đổi sang [Tài liệu tự báo cáo] hoặc [Nhất quán với X] |
| F-05 | P2 | Tracking sai/lỗi thời: KNOWN_ISSUES #6 ("50 FR, 30 mã AC, Mở"), #8 ("37 cần duyệt" so với 15), #12 và NEXT_SESSION:33 "chưa là git repo" (sai: `.git` có 2 commit + remote); TASK_BACKLOG ENV-003 | `git -C AI-Workspace-App rev-parse --show-toplevel` = repo; checklist §4 #13 | KNOWN_ISSUES.md #6,#8,#12; NEXT_SESSION.md:33; TASK_BACKLOG.md:41 | Cập nhật |
| F-06 | P2 | `TESTING.md:11` và `ROADMAP:5` vẫn "AC-001…AC-130"; ACCEPTANCE_CRITERIA.md chưa chứa AC-097..100 | grep | docs/development/TESTING.md:11; docs/IMPLEMENTATION_ROADMAP.md:5 | Ghi "AC-001…130 (có ID dự trữ) + AC-201..433 PROPOSED" |
| F-07 | P2 | DOC_SYNC_LOG không ghi 5 chỉnh sửa đã áp dụng (4 phụ lục D-021 + con trỏ AC) và ghi "22 file" trong khi liệt kê 20 | mtime sau 10:33; grep D-021; đếm file | docs/DOC_SYNC_LOG_PHASE0.md §4 | Thêm DS-30..34, sửa số |
| F-08 | P2 | CC0 sprite vẫn viết như đã xác minh (CF-025 áp dụng một nửa) | `VIRTUAL_OFFICE.md:271` "CC0 1.0 (trang itch.io hiển thị, tác giả xác nhận...)" | docs/VIRTUAL_OFFICE.md:271 | Thêm "[chưa lưu bằng chứng]" như LICENSE_AND_REUSE:352 |
| F-09 | P2 | `--append-system-prompt-file` ghi là liệt kê trong `claude --help`, thực tế chỉ có ký hiệu `[-file]` trong mô tả `--bare` | `claude --help` (grep `-file`: chỉ dòng 55-56 cho cờ này) | DYNAMIC_SKILL_INJECTION.md §4.5 bước 3; PHASE_0_OWNER_DECISION_REGISTER.md D-021 "Bằng chứng" | Hạ nhãn xuống "xuất hiện trong mô tả `--bare`, chưa xác nhận là tùy chọn độc lập"; đưa vào PoC P0 |
| F-10 | P2 | `provider_kind` giữ `mock` ở enum production; DB và PROVIDER_INTEGRATION lệch tập | `DATABASE_DESIGN.md:174`; `PROVIDER_INTEGRATION.md:35` | cả hai | Ghi rõ khi Owner trả lời câu hỏi mock env var |
| F-11 | P2 | REMAINING_RISKS DOC-04 vẫn "Open" dù liên kết đã sửa | grep 4 đường dẫn cũ = chỉ mô tả lịch sử | PHASE_0_REMAINING_RISKS.md:100 | Đóng phần liên kết, giữ phần `VERIFICATION_LOG` |
| F-12 | P3 | `SECURITY.md:574` còn "(644)"; regex xác minh DS-09 quá hẹp | `grep -n 644` | docs/SECURITY.md:574 | Sửa; mở rộng regex |
| F-13 | P3 | Bảng tổng hợp §2 của Register và ADR README chỉ tới D-020; không có ADR cho D-021 | `grep -cE "^\| D-021"` Register = 0 | PHASE_0_OWNER_DECISION_REGISTER.md:18; docs/adr/README.md:5 | Thêm hàng D-021; ADR sau khi Owner quyết |
| F-14 | P3 | 3 cặp FR-AC mà AC gốc không liệt kê FR | node: FR-102/AC-122, FR-127/AC-053, FR-197/AC-045 | TRACEABILITY.md các hàng FR-102/127/197 | Thêm FR vào AC gốc khi hợp nhất |
| F-15 | P3 | Thứ tự cột bảng AC không đồng nhất giữa addendum Phần 1 (FR\|Given...) và Phần 2/3 (Given\|When\|Then\|FR) | đọc AC-201 và AC-301 | traceability_parts/AC_ADDENDUM_PART*.md | Chuẩn hóa khi hợp nhất |
| F-16 | P3 | AC yếu: AC-305 có "không kiểm" trong Then; AC-425 gộp nhiều hành vi và một Q mở | đọc | AC_ADDENDUM_PART2/3 | Tách/viết lại |
| F-17 | P3 | RAM trống ghi 3,9 / 4,18 GB khác nhau giữa file, không có giờ đo | FINAL_REPORT:30; checklist §4 #7; VALIDATION #10 | các file | Ghi giờ đo hoặc "dao động" |

## 10. Chưa kiểm chứng

- Mọi hành vi runtime: Claude SDK `systemPrompt.append`, `--append-system-prompt-file`, `--add-dir` nạp `.claude/skills`, resume snapshot (cần chạy prompt, bị cấm). `codex.exe login status` không chạy lại (tránh chạm trạng thái auth).
- Docker boundary, WSL2, Job Object, emergency stop < 5 s, concurrency 18: không kiểm được (daemon tắt, không chạy prompt).
- Nội dung kỹ thuật của ADR/Register ngoài số đếm và mức bằng chứng; nội dung AC mới chỉ lấy mẫu 18 trong 76.
- Giấy phép (JIK-A-4 CC0, herdr `vendor/`, `assets/sounds`), ToS subscription, visibility remote GitHub.
- Các số web-search (Tauri, React 19.2, Vite 7...) không kiểm lại; chỉ kiểm `npm view` drizzle-orm, phaser, @openai/codex.
- Số liệu live cũ (5 975 ms, 235 MB, 5 process song song): không tái hiện.
- Không kiểm chéo toàn bộ liên kết nội bộ ngoài 4 đường dẫn hỏng đã biết.
