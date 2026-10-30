# DOC_SYNC_LOG_OWR3_A — Đồng bộ tài liệu theo OWR-3 (agent A: security, provider, git, roadmap)

Ngày: 2026-10-30. Nguồn: `owner-responses/OWR-3_2026-10-30_decision_clarification.md` (đã đọc cả phụ lục agent), OWR-2, `DOC_SYNC_LOG_OWR2_A.md`. Phase 0 vẫn mở; Phase 1 chưa được phép. Không mục nào là VERIFIED; chưa có PoC. Đường dẫn tính từ `AI-Workspace-App/docs/` (hai file `PHASE_0_*` ở gốc `AI-Workspace-App/`). Số dòng gần đúng (sau sửa). Xác minh = `grep "OWR-3"` và grep cụm mới có mặt; cụm cũ đã mất khi thay thế. Không chạy git. Mã X-13 (xung đột Tier 1 D-007 và A.10) lấy từ phụ lục OWR-3; chưa thấy trong `PHASE_0_OWNER_DECISION_REGISTER.md` (agent khác sở hữu).

## 1. Đã áp dụng

| Change ID | file:dòng | Cũ (rút gọn) | Mới (rút gọn) | OWR-3 | Xác minh |
|---|---|---|---|---|---|
| B-01 | SECURITY.md:4 | "mặc định ứng dụng 5, Q-07/OWR-2: giới hạn của ứng dụng, không xác nhận subscription" | 5 là giới hạn scheduler NỘI BỘ, không phải giới hạn provider bảo đảm; không 10/18; không chia sẻ credential/seat | A.2, D | grep "scheduler NỘI BỘ" |
| B-02 | SECURITY.md:16 (đoạn 2 mục 0) | mô tả Tier 1 | marker X-13: mâu thuẫn D-007, trỏ §0.6, §0.6 thắng; giữ bản D-007 làm lịch sử | A.10 | grep "MÂU THUẪN VỚI D-007" |
| B-03 | SECURITY.md:~25 (gate) | "Phase 1 chưa được cho phép (OWR-2)" | thêm OWR-3 | mục chung | grep |
| B-04 | SECURITY.md:28-47 (§0.6 mới) | (không có) | chỉ thị sandbox: shell tích hợp không bật trên host; chỉ sandbox execution tool; fail closed; Docker chưa được phép; X-13; ba câu hỏi PENDING (i)(ii)(iii) | A.10, E | grep "0.6 Chỉ thị sandbox" |
| B-05 | SECURITY.md §2.1 | (không có) | marker §2.1-2.3 là Tier 1 D-007 mâu thuẫn A.10 | A.10 | grep "toàn bộ §2.1 đến §2.3" |
| B-06 | SECURITY.md §2.4 | Tier 2 opt-in | marker OpenHands hoãn, Tier 2 không thuộc MVP | A.10 | grep "Tier 2 KHÔNG thuộc MVP" |
| B-07 | SECURITY.md §5.3 (~dòng 425) | `remind` 24 h, 15 phút | tách `remind_at` / `expires_at`; WAITING_APPROVAL không tính slot khi giải phóng worker + persist | A.12, A.15, X-7 | grep "remind_at" |
| B-08 | SECURITY.md R1 | R1 Tier 1 | marker X-13 R1 phụ thuộc câu trả lời (i)/(ii) | A.10 | grep "R1 là hệ quả của D-007" |
| B-09 | security/SANDBOX_DESIGN.md:7-8 (banner) | (không có) | banner chỉ thị A.10, X-13, 3 PENDING; §4.7 hook-rewrite Bash bị thay bởi tool `sandbox.exec`; Tier 1 §4 lịch sử | A.10, E | grep "CHỈ THỊ SANDBOX CHẶT HƠN" |
| B-10 | security/SANDBOX_DESIGN.md §4.5 | "Owner tự khởi động Docker Desktop" | thay bằng "Owner chưa cho phép; chờ Owner cho phép" | E | grep "chờ Owner cho phép" |
| B-11 | security/SANDBOX_DESIGN.md §5 | Tier 2 | marker hoãn khỏi MVP | A.10 | grep "Tier 2 KHÔNG thuộc MVP" |
| B-12 | security/SANDBOX_DESIGN.md §9 | Docker tắt -> deny; agent vẫn đọc/ghi | thêm fail closed, task BLOCKED, marker PENDING (i) về ghi file | A.10 | grep "fail closed" |
| B-13 | security/PERMISSION_MODEL.md §4.1 (~dòng 52-54) | Q-03 note | thêm A.13 L0-L5 chuẩn; marker A.10/X-13 cho Edit/Write/Bash host | A.13, A.10 | grep "OWR-3 A.13" |
| B-14 | security/PERMISSION_MODEL.md §6.2 bảng (`expires_at`), §6.4 | `expires_at` 7 ngày marker | tách `remind_at`/`expires_at`, 540 s (X-7) | A.12, X-7 | grep "remind_at" |
| B-15 | security/PERMISSION_MODEL.md ~dòng 317 | "giới hạn hook 600 s" | 540 s (X-7), đã chốt | X-7 | grep "600 s cũ" |
| B-16 | security/THREAT_MODEL.md:15 | "tối đa 18 ... mặc định 5, trần 18" | thêm 5 nội bộ; marker X-13 TB2/TB4 | A.2, A.10 | grep |
| B-17 | PROVIDER_INTEGRATION.md:15 | MVP chỉ Claude (OWR-2) | thêm OWR-3 A.8: mục Codex/OpenHands là tham khảo sau MVP | A.8 | grep "OWR-3 A.8" |
| B-18 | PROVIDER_INTEGRATION.md §7 bảng Options (hàng mới sau PreToolUse) | PreToolUse + hook | hàng: shell tích hợp không bật trên host, sandbox execution tool, 3 PENDING, xung đột D-007/X-13 | A.10 | grep "disallowedTools" |
| B-19 | PROVIDER_INTEGRATION.md §13 (~334-339) | Q-07/OWR-2; Claude 5, Codex 3 | 5 là nội bộ; Owner statement chưa xác minh độc lập; không chia sẻ credential/seat; hàng Claude 5 và Codex 3 gắn "KHÔNG phải số provider cho phép", Codex hoãn | A.2, D, A.8 | grep "Owner statement" |
| B-20 | PROVIDER_INTEGRATION.md §17 (~416) | 540 s, 24 h, 15 phút | thêm remind_at/expires_at, A.15 | A.12, A.15, X-7 | grep |
| B-21 | PROVIDER_INTEGRATION.md §24, §25 (mục 2, 5), §20 live smoke | Tier 1 mitigation; CF-048 mở; "5 session song song" | marker X-13; CF-048 giải quyết A.15; 5 song song chỉ kiểm tra kỹ thuật, không là xác nhận điều khoản | A.10, A.15, D | grep "CF-048" |
| B-22 | architecture/RESOURCE_MANAGEMENT.md:48-49 | Q-07/OWR-2; claude 5, codex 3 | 5/3 nội bộ, Owner statement, không chia sẻ credential/seat | A.2, D | grep |
| B-23 | architecture/RESOURCE_MANAGEMENT.md:55 | marker CF-048 CẦN QUYẾT ĐỊNH | thay bằng giải quyết A.15 (điều kiện giải phóng worker + persist; approval còn giữ process vẫn chiếm slot; quota vẫn áp dụng) | A.15 | grep "OWR-3 A.15" |
| B-24 | architecture/RESOURCE_MANAGEMENT.md:104, §10 comment, §11 note | approval 540 s, 24 h/15 phút | tách `remind_at`/`expires_at`; comment 5 nội bộ | A.12, A.2 | grep |
| B-25 | architecture/AGENT_SCHEDULER.md:151, :161 | "chờ slot global (5/5)"; WAITING_APPROVAL | note 5 nội bộ; A.15 và A.12 | A.2, A.15, A.12 | grep "OWR-3" |
| B-26 | architecture/AGENT_COLLABORATION.md:56 | "L4 hoặc chuyển provider" | A.13 L0-L5; làm rõ "L4 command" là nhãn cũ (PENDING); A.12 | A.13, A.12 | grep "OWR-3 A.13" |
| B-27 | architecture/WORKSPACE_AND_GIT_ISOLATION.md §7.1 (marker, dòng ~111), §7.2 | JSON `Bash(...)` allow + hook Bash trên host; mount ~/.claude | marker A.10/X-13: các mục Bash host không dùng; 3 PENDING; git cũng là thực thi tiến trình; mount ~/.claude trái D-008 | A.10 | grep "X-13" |
| B-28 | architecture/INFRASTRUCTURE_DESIGN.md §6.1, §8, §4.2 (staging) | Sandbox Executor ephemeral; staging worker giữ claude-home | Sandbox Executor là đường duy nhất chạy code agent; fail closed khi Docker down; staging trái D-008/A.10 | A.10 | grep "X-13" |
| B-29 | operations/AGENT_OPERATIONS.md:21-22, ~80, ~142 | executing default 5/18; checklist 540 s | 5 nội bộ; A.15; remind_at/expires_at; "L4 command" PENDING; Codex sau MVP | A.2, A.15, A.12, A.13, A.8 | grep |
| B-30 | development/ENVIRONMENT_SETUP.md:~209, ~302 | Codex hoãn; Owner tự mở Docker | A.8 và E: chờ Owner cho phép; không test agent thật; mục Codex không thuộc MVP | A.8, E | grep |
| B-31 | development/TESTING.md:~81, ~88 | P-D Codex hoãn | A.8; E: Docker/VAL/27 runbook/PoC Conductor/test agent thật chưa được phép; test fail-closed chưa chạy | A.8, E, A.10 | grep |
| B-32 | development/CI_CD.md:~168 | D-005 note | A.8; A.10: CI không cho agent chạy code trên host runner | A.8, A.10 | grep |
| B-33 | development/DEPLOYMENT.md:159 | "chạy task thật nhỏ" | marker: chỉ sau sandbox + gate; Docker chưa phép | A.10, A.11, E | grep |
| B-34 | IMPLEMENTATION_ROADMAP.md (Phase 2 mục 3.7, mở đầu) | "Adapter Claude Code thật hoạt động" | note A.11: Phase 2 mock/sandbox; "thật/live" chỉ sau gate; P2-10, P3-12, P5-13 gắn nhãn; P2-13 cần phép riêng | A.11, E | grep "OWR-3 A.11" |
| B-35 | IMPLEMENTATION_ROADMAP.md EXT-3 | Docker "Chưa chạy" | Owner chưa cho phép; Docker là điều kiện tiên quyết cứng | A.10, E | grep |
| B-36 | IMPLEMENTATION_ROADMAP.md "Rủi ro chấp nhận có điều kiện" | overlay qua Bash host | marker X-13 PENDING | A.10 | grep |
| B-37 | IMPLEMENTATION_ROADMAP.md IR-2 | 5->10->18 | 5 nội bộ; không tăng khi chưa xác minh | A.2, D | grep |
| B-38 | IMPLEMENTATION_ROADMAP.md cuối file (Phụ lục OWR-3) | (không có) | 6 mục: A.11 (đối chiếu: mục 4 và 5 phụ lục OWR-2 đã khớp), A.10, A.2/D, A.8, A.12/A.15/X-7, phạm vi mục E | A.8, A.10-A.12, A.15, D, E | grep "Phụ lục OWR-3" |
| B-39 | PROJECT_RISKS.md R-TOS-QUOTA, R-WIN-SANDBOX, ghi chú D-007 | Q-07/OWR-2; Tier 2 mitigation | OWR-3 D wording + không chia sẻ seat + no API key fallback; marker X-13 | D, A.10 | grep |
| B-40 | research/MULTI_AGENT_SCALABILITY.md:12, §8 mục 3 | "5 là mức cho phép quan sát" | 5 là mức thận trọng nội bộ; note OWR-3 D, A.15 | A.2, D, A.15 | grep |
| B-41 | research/CLAUDE_CODE_INTEGRATION.md:10+, :488 | 10 mặc định cũ, đã có note OWR-2 | note OWR-3: 5 nội bộ; MVP chỉ Claude; thí nghiệm Bash host là lịch sử; không chạy lại | A.2, D, A.8, A.10, E | grep |
| B-42 | research/CODEX_INTEGRATION.md:~8, :533 | cap Codex 3, global 5 | note A.8 + A.2/D: số nội bộ, không phải OpenAI cho phép; Codex sau MVP | A.8, A.2, D | grep |
| B-43 | poc/POC_DOCKER_SANDBOX.md:3-8, §3, §6 | BLOCKED, chờ Owner tự mở Docker | BLOCKED / NOT AUTHORIZED, chờ Owner cho phép; giữ baseline (daemon tắt, RAM 4,04/15,77 GB); mục 3 thành "chưa được phép" | E | grep "NOT AUTHORIZED" |
| B-44 | ../PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md (mục cuối "OWR-3") | (không có) | sandbox directive, hệ quả, PENDING, provider terms, approval; mọi ô Owner sign-off vẫn trống | A.10, D, E | grep "## OWR-3" |
| B-45 | ../PHASE_0_REMAINING_RISKS.md (mục cuối "sau OWR-3") | (không có) | RR-OWR3-01..05 | A.10, A.15, D, E | grep "RR-OWR3" |

X-7 (540 s) và X-8 (audit 365 ngày): đã quét `grep 540|365|180|600 s`; đã nhất quán ở SECURITY, PERMISSION_MODEL (sửa B-15), PROVIDER_INTEGRATION, RESOURCE_MANAGEMENT, AGENT_OPERATIONS, AUDIT_LOGGING, DEVSECOPS_GUIDE. Các "180"/"600 s" còn lại là trích nguồn lịch sử, mô tả giá trị bị thay hoặc docs ngoài (SECURITY §refs "timeout 600 s" của Claude Code hooks là sự kiện docs, không phải giá trị dự án).

## 2. Conflicts found

| # | Vị trí | Nội dung | Vì sao không tự sửa | Ai quyết định |
|---|---|---|---|---|
| X-13 | SECURITY §0.6, §2.1-2.4, §3, §5, R1/R4; SANDBOX_DESIGN §4/§4.7/§5; WORKSPACE §7.1/7.2; PROVIDER §7 Options, §17, §24; INFRA §6.1, §4.2; PERMISSION_MODEL §4.1 bảng; ROADMAP | D-007 (APPROVED có điều kiện: CLI/agent trên host + hook; Docker cho build/test/run) mâu thuẫn A.10 (không shell host; chỉ sandbox; Docker-only chưa đủ; fail closed) | Owner chưa quyết; marker đã đặt; thiết kế cũ giữ lịch sử | Owner |
| OQ-i | cùng trên | Host Edit/Write trong worktree có được phép không | A.10 không nêu | Owner |
| OQ-ii | cùng trên | SDK runtime trên host (giữ login) có là host execution; nếu có thì xung đột D-008 | A.10 không nêu | Owner |
| OQ-iii | cùng trên | Runtime sandbox thay OpenHands: executor Docker tự xây chưa có task trong roadmap; Docker là điều kiện tiên quyết cứng nhưng chưa phép khởi động | Quyết định phạm vi | Owner |
| C-1 | AGENT_COLLABORATION §4 dòng 56, AGENT_OPERATIONS (Approve/Deny): "L4 command" | Nhãn cũ "L4" có thể bị nhầm với permission level L4 Operator (A.13) | Không rõ ý nghĩa gốc | Architect/PM |
| C-2 | AGENT_ORGANIZATION.md:98 (agent B) | Bảng L0..L4 vs L0..L5 (đã có marker OWR-2, chưa giải quyết) | Thuộc agent B; báo cáo | Agent B |
| C-3 | RESOURCE_MANAGEMENT §4.1 / PROVIDER §17 | A.15 nói không tính slot "nếu worker đã giải phóng và persist"; approval mức tool trong 540 s còn giữ process nên vẫn chiếm slot. Đã ghi điều kiện, chưa quyết cơ chế (hủy session rồi resume hay giữ) | Cần thiết kế/PoC; không chạy PoC (mục E) | Architect/Owner |
| C-4 | PERMISSION_MODEL §6.4, DATABASE_DESIGN | `expires_at` 7 ngày / `timeout_sec` 3600 cho approval thường vs A.12 (chỉ 15 phút cho 3 hành động); hạn tối đa approval thường chưa quy định (X-3 OWR-2 còn mở); cột `remind_at` chưa có trong schema | DATABASE_DESIGN thuộc agent khác | Owner / agent DATABASE |
| C-5 | OWR-3 phụ lục #2 vs `PHASE_0_OWNER_DECISION_REGISTER.md` | Mã X-13 chưa thấy trong register tại thời điểm viết | Register thuộc agent khác | Agent register |
| C-6 | ROADMAP P2-11/12, P4-11, P5-14, P7-09, P7-01 "chứng minh 18" | Vẫn nằm trong bảng task dù Codex hoãn, 18 không tuyên bố; đã có Phụ lục | Xếp lại task do Owner duyệt khi mở Phase 1 | Owner |
| C-7 | SECURITY mục 8 (R-N, §5 hook HTTP, `--permission-mode manual`) | Vẫn mô tả raw CLI/host Bash (X-5 OWR-2) | Viết lại cần PoC; marker | Architect |

## 3. Not synced and why

| File / hạng mục | Lý do |
|---|---|
| Viết lại thân SECURITY §2.2-2.3, §3, §5 và SANDBOX_DESIGN §4 theo mô hình "sandbox execution tool" | Owner chưa trả lời (i)-(iii); chỉ đặt quy tắc mới (§0.6) + marker, giữ thiết kế D-007 làm lịch sử; không tự quyết định |
| PERMISSION_MODEL cột "Tool mặc định"/"Container exec" theo từng L0-L5 | Phụ thuộc OQ-i; chỉ ghi marker |
| security/CREDENTIAL_MANAGEMENT.md, AUDIT_LOGGING.md, operations/DEVSECOPS_GUIDE.md | Đã nhất quán X-7/X-8; không có thay đổi OWR-3 cần thiết |
| development/DEVELOPMENT_GUIDE.md, CODING_STANDARDS.md | Quét, không thấy mâu thuẫn trực tiếp |
| USER_ROLES_AND_PERMISSIONS.md, WORKFLOW_ENGINE.md, DATABASE_DESIGN.md, AGENT_ORGANIZATION.md, DECISION_LOG/Register, ADR | Không thuộc agent A |
| Mọi trạng thái VERIFIED / Phase 0 completed / Phase 1 started | Không đánh dấu |
| Ô Owner sign-off trong PHASE_0_SECURITY_SIGNOFF_CHECKLIST | Giữ trống |
| Chạy PoC, Docker, git | Không được phép (OWR-3 mục E, yêu cầu công việc) |
| SYNC-OWR3-ORCH-01 | docs/IMPLEMENTATION_ROADMAP.md:267 | `L0-L4` đổi thành `L0-L5` | OWR-3 A.13 | Orchestrator | grep xác nhận |
| SYNC-QA3-01 | docs/PROVIDER_INTEGRATION.md:123,130,155,260,331,344,356 | `session.state_changed`, `rate_limit.updated`, `provider.status_changed` đổi thành dotted không gạch dưới | OWR-3 A.16, QA3 F3-02 | Orchestrator | grep |
