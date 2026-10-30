# DOC_SYNC_LOG_OWR2_A — Đồng bộ tài liệu theo OWR-2 (agent A: security, provider, git, roadmap)

Ngày: 2026-10-30. Nguồn: `owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Phase 0 vẫn mở; Phase 1 chưa được phép. Không mục nào dưới đây được đánh dấu VERIFIED; chưa có PoC. Đường dẫn tính từ `AI-Workspace-App/docs/`. Số dòng là vị trí sau khi sửa (gần đúng). Xác minh = grep cụm mới có mặt (và cụm cũ đã mất khi thay thế). Không chạy git.

## 1. Đã áp dụng

| Change ID | file:dòng | Cũ (rút gọn) | Mới (rút gọn) | Quyết định | OWR-2 | Xác minh |
|---|---|---|---|---|---|---|
| A-01 | SECURITY.md:4-5 | "tới 10 AI employee", Claude Code CLI hoặc Codex | mặc định 5 là giới hạn ứng dụng; adapter chính Agent SDK; Codex hoãn | Q-07, Q-17, D-005 | §2, §1 | grep "Q-17/D-004/OWR-2" |
| A-02 | SECURITY.md:25 | (không có) | mục 0.5 Security gates (a)-(g), không mount credential vào sandbox, Phase 0 mở | D-008, D-019, §8 | §3, §8 | grep "Security gates (OWR-2" |
| A-03 | SECURITY.md:22 | không bao giờ đọc credential | thêm "(D-008)" | D-008 | §3 | grep |
| A-04 | SECURITY.md:111 | (không có) | ghi chú sơ đồ §2.3 là fallback raw CLI | Q-17, CF-014 | §1 | grep "sơ đồ dưới minh họa biến thể raw CLI" |
| A-05 | SECURITY.md:~94 | `--permission-mode manual` | gắn nhãn fallback; mặc định `default` + whitelist | Q-17, CF-014 | §1 | grep "biến thể raw CLI = fallback" |
| A-06 | SECURITY.md:169 | 10 agent ≈ 3–4 project | thêm chú thích không phải đề xuất concurrency | Q-07 | §2 | grep |
| A-07 | SECURITY.md:205 | §2.5 Codex-specific | thêm ghi chú Codex hoãn | D-005, Q-10 | §3 | grep |
| A-08 | SECURITY.md:352-361 (§5.1) | PreToolUse hook là nguồn chính, Codex, app-level | viết lại: SDK `canUseTool` + PreToolUse in-process (một policy engine); mode `default` + whitelist; CLI hook = fallback; Codex hoãn; merge `integration/*` vs `main` | Q-17, D-004, CF-013, CF-014, Q-04 | §1, §2 | grep "canUseTool" §5.1 |
| A-09 | SECURITY.md:~399 (§5.3 source) | `claude_hook\|claude_control\|codex\|app` | `claude_sdk\|claude_hook[fallback]\|...\|codex[hoãn]\|app` | Q-17, D-005 | §1, §3 | grep "claude_sdk" |
| A-10 | SECURITY.md:403 (§5.3) | hook giữ HTTP 600/540 s, hết hạn deny, task `blocked` | chờ tối đa 540 s, lưu pending, `WAITING_HUMAN`, không giữ worker, nhắc 24 h, 3 hành động nhạy cảm riêng, hết hạn 15 phút | Q-05, CF-007 | §2 | grep "540 giây" |
| A-11 | SECURITY.md:502 (§8) | Retention 180 ngày audit | audit online 365 ngày + signed export 2 năm | Q-06, CF-006 | §2 | grep "365 ngày" SECURITY.md |
| A-12 | SECURITY.md R10, M1, M3 | Codex chưa cài; HTTP PreToolUse hook; Codex adapter | thêm nhãn hoãn D-005; M1 dùng SDK canUseTool + PreToolUse in-process | D-005, Q-17 | §3, §1 | grep |
| A-13 | PROVIDER_INTEGRATION.md:15 | Chỉ Claude Code, Codex, OpenHands, Generic CLI | thêm phạm vi MVP: Claude SDK; Codex hoãn; OpenHands sau Phase 4 và ACP tắt; Phase 2 mock/sandbox khi gate chưa đạt | Q-17, D-005, D-006, Q-02 | §1-§3, §8 | grep "OWR-2 2026-10-30: MVP chỉ Claude" |
| A-14 | PROVIDER_INTEGRATION.md:192 | Raw CLI chỉ healthcheck/fallback | thêm: SDK adapter chính; phân biệt subscription auth, API auth, usage limits; SDK không đồng nghĩa không giới hạn session; ràng buộc provider contract | Q-17, §5.1 | §1, §5.1 | grep "Q-17/OWR-2" |
| A-15 | PROVIDER_INTEGRATION.md:217 (§8), :242 (§9) | tiêu đề CodexAdapter, OpenHandsAdapter | thêm ghi chú hoãn | D-005, D-006, Q-02, Q-10 | §3 | grep |
| A-16 | PROVIDER_INTEGRATION.md:334 | global mặc định 5, tối đa 18 — D-010 | thêm Q-07: không phải xác nhận subscription, không tự tăng 10/18, không tuyên bố 18 đã kiểm thử | Q-07 | §2, §8 | grep "KHÔNG xác nhận subscription" |
| A-17 | PROVIDER_INTEGRATION.md:343, :374 | auto-switch mặc định tắt [Cần phê duyệt]; QUOTA_EXHAUSTED queue | không auto-failover, thông báo Owner và chờ quyết định | Q-08 | §2 | grep "Q-08/OWR-2" |
| A-18 | PROVIDER_INTEGRATION.md:415 | tool-level 10 phút → BLOCKED | 540 s → pending, WAITING_HUMAN, nhắc 24 h, 15 phút cho 3 hành động | Q-05, CF-007 | §2 | grep "thống nhất mọi tài liệu" |
| A-19 | PROVIDER_INTEGRATION.md:479 (P4) | MVP chỉ native + LLM key, Cần phê duyệt | hoãn đến sau Phase 4, ACP tắt; theo Q-02/D-006, chưa VERIFIED | Q-02, D-006 | §2, §3 | grep "Q-02/D-006" |
| A-20 | architecture/RESOURCE_MANAGEMENT.md:104 | approval_timeout 24 h → QUEUED | approval_wait_timeout 540 s → WAITING_HUMAN, 24 h chỉ nhắc, 15 phút cho 3 hành động | Q-05, CF-007 | §2 | grep "approval_wait_timeout" |
| A-21 | architecture/RESOURCE_MANAGEMENT.md:48, :132, :190 | default 5, max 18; không tự chuyển provider; kết luận 18 khả thi | thêm Q-07 (không xác nhận ToS, không tự tăng, không tuyên bố đã kiểm thử); thêm Q-08 | Q-07, Q-08 | §2 | grep "Q-07/OWR-2" |
| A-22 | architecture/RESOURCE_MANAGEMENT.md:55 | PAUSED, WAITING_APPROVAL không chiếm slot | giữ nguyên, thêm marker `[CẦN QUYẾT ĐỊNH, CF-048/Q-05]` | Q-05 | §2 | grep "CF-048/Q-05" |
| A-23 | research/MULTI_AGENT_SCALABILITY.md:12, :124 | (không có) | hai ghi chú Q-07 (không xác nhận ToS, không tự tăng 10/18, không tuyên bố 18 đã kiểm thử); giữ "chưa xác minh ToS" cũ | Q-07 | §2, §8 | grep "Q-07/OWR-2" |
| A-24 | security/PERMISSION_MODEL.md:48 | cần mở rộng permission_profile [Cần phê duyệt] | thêm ghi chú Q-03: L0-L5, `permission_level` + `permission_profile`, không hỗ trợ `full` | Q-03, CF-008 | §2 | grep "Q-03, OWR-2" |
| A-25 | security/PERMISSION_MODEL.md:205 (§6.4) | hết 540 s deny pending_human; expires_at 7 ngày | 540 s, WAITING_HUMAN, nhắc 24 h, 15 phút; marker cho expires_at 7 ngày/3600 s | Q-05, CF-007 | §2 | grep "Q-05, OWR-2" |
| A-26 | security/PERMISSION_MODEL.md:~313 | Lead không approve [Cần phê duyệt] | Q-04: Lead không thay Owner; main luôn Owner; `integration/*` sau gate | Q-04 | §2 | grep "Q-04 (OWR-2" |
| A-27 | security/PERMISSION_MODEL.md:~345 | câu hỏi canUseTool vs PreToolUse | thêm trả lời Q-17: SDK chính, hai tầng một engine, hook HTTP fallback | Q-17, D-004 | §1 | grep "Q-17/D-004, OWR-2" |
| A-28 | architecture/WORKSPACE_AND_GIT_ISOLATION.md:~109 | thiết kế acceptEdits + settings.json + hook command | giữ nguyên + marker `[CẦN QUYẾT ĐỊNH, CF-014/Q-17/D-004]`; no credential/Docker socket mount; fail closed | Q-17, CF-014, §8 | §1, §8 | grep "CF-014/Q-17/D-004" |
| A-29 | architecture/WORKSPACE_AND_GIT_ISOLATION.md:~134 | Codex CLI (khi cài) | thêm hoãn D-005 | D-005 | §3 | grep |
| A-30 | architecture/WORKSPACE_AND_GIT_ISOLATION.md:~161 (§8) | integration → main chỉ khi user Approve | thêm Q-04/D-009: agent merge `integration/*` sau gate; main luôn Owner approval 15 phút; Lead không thay Owner; mỗi task một worktree/branch, review diff, không auto-merge main | Q-04, D-009, Q-05 | §2, §3 | grep "Q-04, D-009, OWR-2" |
| A-31 | architecture/AGENT_COLLABORATION.md:56 | L4 hoặc chuyển provider | thêm Q-08 và Q-05 | Q-08, Q-05 | §2 | grep |
| A-32 | operations/AGENT_OPERATIONS.md:115, :142, :171 | chuyển Codex bằng approval; SLA 24 h; Không auto-switch [Cần phê duyệt] | Q-08 không auto-failover; Q-05 540 s/`WAITING_HUMAN`/24 h nhắc/15 phút; Codex hoãn | Q-08, Q-05, D-005 | §2, §3 | grep "Q-08/OWR-2" |
| A-33 | security/AUDIT_LOGGING.md:158, :239, :265 | Đề xuất; Cần phê duyệt (SECURITY nói 180) | Q-06 đã chốt, SECURITY §8 đã sửa | Q-06 | §2 | grep "Q-06" |
| A-34 | operations/DEVSECOPS_GUIDE.md:170 | [Cần phê duyệt] | Q-06: 365 ngày online + signed export 2 năm | Q-06 | §2 | grep "Q-06" |
| A-35 | security/CREDENTIAL_MANAGEMENT.md:204 | Đề xuất (D-008 chờ duyệt) | D-008 theo OWR-2, thêm "không mount thư mục credential của Owner vào Agent sandbox" | D-008 | §3 | grep "không mount thư mục credential" |
| A-36 | security/SANDBOX_DESIGN.md:141 (§4.5) | (không có) | ghi chú D-008: không mount credential/Docker socket; fail closed; Docker cần cho validation; quy trình Owner khởi động Docker | D-008, Q-10, §8, §5.3 | §3, §5.3, §8 | grep "D-008, OWR-2 mục 8" |
| A-37 | development/ENVIRONMENT_SETUP.md:30, :207, :246 | Codex CLI cài/kiểm tra | nhãn tùy chọn/hoãn; không chạy `npm i -g @openai/codex`; quy trình Docker thủ công | D-005, Q-10 | §3, §5.2, §5.3 | grep "TÙY CHỌN / HOÃN" |
| A-38 | development/TESTING.md:88, :103, :210 | P-D chặn tới khi Owner cài Codex; bộ test 18 agent | hoãn D-005; ghi chú chưa chạy, không tuyên bố 18 đã kiểm thử, chỉ 3 PoC được phép | D-005, Q-07, §5.6 | §3, §5.6, §8 | grep "OWR-2 2026-10-30, Q-07" |
| A-39 | development/CI_CD.md:168 | Agent CLI trong CI | thêm ghi chú: CI không cài Codex; SDK chính; mock chỉ CI | D-005, Q-17 | §3, §1 | grep "D-005/OWR-2: Codex hoãn" |
| A-40 | IMPLEMENTATION_ROADMAP.md:562+ (Phụ lục) | 4 điều kiện cổng vào Phase 2 (lời lỏng hơn) | viết lại theo OWR-2 mục 8 (9 mục): Phase 2 chỉ mock/sandbox khi gate chưa đạt; không tuyên bố 18; D-019 Phase 3; Codex/OpenHands hoãn; Phase 0 mở, Phase 1 chưa được phép | §8, Q-07, D-019, D-005, D-006 | §8 | grep "Security gates và phạm vi phase" |
| A-41 | IMPLEMENTATION_ROADMAP.md (EXT-2, IR-8) | Cài Codex là cổng; IR-8 [Cần phê duyệt] | EXT-2 HOÃN; IR-8 theo D-005 | D-005 | §3 | grep "HOÃN, D-005" |
| A-42 | PROJECT_RISKS.md:58, :97 | tăng 5→10→18 khi có bằng chứng; Codex chưa cài | Q-07 wording; D-005/Q-10 hoãn | Q-07, D-005 | §2, §3 | grep "Q-07/OWR-2" PROJECT_RISKS.md |
| A-43 | research/CLAUDE_CODE_INTEGRATION.md:~7 | (không có) | ghi chú có ngày: SDK chính, 3 loại auth/limit, 10 session cũ không phải xác nhận ToS, không failover, 540 s | Q-17, Q-07, Q-08, Q-05 | §1, §2 | grep "Ghi chú 2026-10-30" |
| A-44 | research/CODEX_INTEGRATION.md:~8 | (không có) | ghi chú có ngày: Codex hoãn, không cài | D-005, Q-10 | §3, §5.2 | grep "Ghi chú 2026-10-30" |

## 2. Conflicts found

| # | Vị trí | Nội dung xung đột | Vì sao không tự sửa | Cần ai quyết định |
|---|---|---|---|---|
| X-1 | WORKSPACE_AND_GIT_ISOLATION §7.1 (acceptEdits, `.claude/settings.json` trong worktree, hook `command`, `claude -p`) vs Q-17/CF-014 | Thiết kế mâu thuẫn: Agent SDK, mode `default` + whitelist, không ghi settings trong worktree | Cần viết lại cả khối JSON, script hook, mô hình enforcement, interface `prepareAgentConfig`; marker đã đặt | Tác giả doc/Architect |
| X-2 | RESOURCE_MANAGEMENT §4.1 dòng 55 (WAITING_APPROVAL không chiếm slot) vs PROVIDER_INTEGRATION §17 (approval mức tool giữ process, chiếm slot) vs CF-048 | OWR-2 chỉ nói quá 540 s thì không giữ worker; không nói trong 540 s có chiếm slot không | Cần quyết định | Owner/Architect |
| X-3 | PERMISSION_MODEL §6.4 `expires_at` mặc định 7 ngày, `onTimeout='reject'`; DATABASE_DESIGN `timeout_sec` 3600, `onTimeout reject` | OWR-2 chỉ có 540 s, nhắc 24 h, 15 phút cho 3 hành động; không nói hạn tối đa approval thường | OWR-2 không quy định; DATABASE_DESIGN không thuộc A | Owner; agent sở hữu DATABASE_DESIGN |
| X-4 | PERMISSION_MODEL §4.1: enum `permission_level` (`read_only/standard/elevated/full`) và cách mã hóa L0-L5 | Q-03 cấm `full` và dùng `permission_level` + `permission_profile`, nhưng chưa chốt bỏ `full` khỏi enum hay CHECK, và L5 `elevated` + `profile=orchestrator` | Chi tiết schema thuộc DATABASE_DESIGN; mapping trong bảng giữ nguyên | Architect |
| X-5 | SECURITY §2.3 sơ đồ, §3, §8 (HTTP hook, `--settings`, `--permission-mode manual`), R8 | Còn mô tả raw CLI làm kiến trúc nhiều chỗ | Chỉ thêm nhãn fallback; viết lại toàn bộ cần PoC SDK | Architect/Security |
| X-6 | IMPLEMENTATION_ROADMAP: Phase 7 mục tiêu và P7-01/EC-7.x "chứng minh 18 đồng thời"; P2-13 benchmark 5/10/18; P2-11/12, P4-11, P5-14 Codex; P5-02 projection Codex; P7-09 OpenHands "post-MVP" | OWR-2: không tăng 10/18 khi chưa xác minh ToS; Codex hoãn; OpenHands sau Phase 4 | Chưa gỡ task; Phụ lục ghi rõ ưu tiên; cần Owner duyệt xếp lại khi mở Phase 1 | Owner/Delivery Lead |
| X-7 | SECURITY §0.1, §0.4 (ví dụ "10 agent", T9 "10 agent cùng npm run build"), RESOURCE_MANAGEMENT §11 bảng 5/10/18, MULTI_AGENT_SCALABILITY nội dung | Các số 10/18 là ví dụ/ngoại suy | Giữ làm lịch sử, đã thêm chú thích Q-07 | Không cần thêm trừ khi Owner muốn xóa |
| X-8 | RESOURCE_MANAGEMENT/PROVIDER_INTEGRATION: trần cấu hình `max 18` (D-010 APPROVED) vs Q-07 "không tự tăng lên 10/18 khi chưa xác minh" | Trần cấu hình 18 vẫn là giá trị có thể đặt; chưa rõ có giữ trần hay hạ | Chỉ thêm ghi chú | Owner |
| X-9 | PERMISSION_MODEL §5 ví dụ hook HTTP dòng ~148 | Ví dụ HTTP hook, fallback | Chỉ ghi chú ở mục câu hỏi mở | Security |
| X-10 | SECURITY §5.2 `ttlMinutes` default 30, max 480 và PERMISSION_MODEL cached decision `scope=session/project` vs Q-05 | Q-05 chỉ nêu 15 phút cho 3 hành động; TTL khác cho hành động thường chưa được OWR-2 quy định | Giữ nguyên | Owner |

## 3. Not synced and why

| File / hạng mục | Lý do |
|---|---|
| security/USER_ROLES_AND_PERMISSIONS.md (Q-03 L0-L5, Q-04 POL-11/12, §5.5 "không timeout, nhắc 24 h") | Không thuộc agent A; báo cáo để agent sở hữu đồng bộ |
| docs/WORKFLOW_ENGINE.md (timeout approval, approverRoles) | Thuộc agent B (Q-05); báo cáo |
| docs/DATABASE_DESIGN.md (`timeout_sec` 3600, `permission_level` enum, `provider_configs.max_concurrent`) | Không thuộc agent A; xem X-3, X-4 |
| docs/security/THREAT_MODEL.md | Không có thay đổi cần thiết ngoài phạm vi đã quét; không sửa |
| architecture/AGENT_SCHEDULER.md, architecture/INFRASTRUCTURE_DESIGN.md, operations/DEVSECOPS_GUIDE.md (ngoài Q-06), development/DEPLOYMENT.md, DEVELOPMENT_GUIDE.md, CODING_STANDARDS.md | Quét, không thấy giá trị trực tiếp xung đột OWR-2 ngoài số 18 minh họa; không sửa |
| Các hàng P2-11/P2-12, P4-11, P5-14, P2-13, P7-09 trong ROADMAP | Xem X-6: cần Owner duyệt xếp lại |
| SECURITY §2.2/§2.3/§3 toàn bộ viết lại theo SDK | Xem X-5: cần PoC SDK; chưa VERIFIED |
| Mọi trạng thái VERIFIED, Phase 0 completed, Phase 1 started | Không đánh dấu; OWR-2 §7, §8 |
