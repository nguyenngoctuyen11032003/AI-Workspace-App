# QA & Technical Review — Phase 0 (AI WORKSPACE)

Reviewer độc lập: QA & Technical Reviewer · Ngày: 2026-10-30 · Phạm vi: chỉ tài liệu (documentation-only), không sửa file nào khác.
Phương pháp: chạy lệnh thật (`wc -l`, `grep`, `find`, `git log`, `claude --version`, `claude auth status`, `codex.exe --version`, `npm view`, node script đối chiếu FR↔AC và liên kết tài liệu). Không tin tuyên bố tự thân của tài liệu. Không đọc nội dung file credential (chỉ liệt kê tên/kích thước file).
Nhãn: **[Đã kiểm]** = tôi tự chạy lệnh/đối chiếu; **[Không kiểm được]** = ghi rõ ở §7.

---

## 0. Tóm tắt điều hành

- **Verdict chung: PASS WITH FIXES.** Bộ tài liệu đầy đủ về số lượng, nội dung thật (không filler), tự nhận diện hầu hết mâu thuẫn trong bảng "Mâu thuẫn" cuối mỗi tài liệu, và tuân thủ chính sách credential/AGPL/không auto-deploy.
- **Không có tài liệu nào ghi "Đã duyệt" cho thứ chủ sở hữu chưa duyệt** [Đã kiểm]. `DECISION_LOG.md` ghi rõ "Chờ duyệt" cho D-001…D-020 (riêng dòng D-007 bị thiếu cột Phê duyệt, xem P2-04).
- **P0 = 2** (traceability Acceptance Criteria bị hỏng; enum `provider_kind` mâu thuẫn trực tiếp giữa DB và Provider contract). P1 = 14, P2 = 9.
- Yếu điểm chính: (a) tài liệu "chính tắc" đã hoà giải mâu thuẫn nhưng **tài liệu nguồn không được vá**, người triển khai đọc nhầm tài liệu sẽ lấy giá trị sai (audit table, retention, approval timeout, ORM trong ownership manifest); (b) 50/199 FR không có AC truy vết được; (c) thiết kế Windows (SDK + hook, Docker/WSL, 18 session) mới được kiểm chứng ở mức nhẹ.
- Hạng mục hoàn tất Phase 0 còn thiếu tại thời điểm rà: `docs/adr/` rỗng, `NEXT_SESSION.md` và `PHASE_0_FINAL_REPORT.md` chưa tồn tại (xem §1.4; đang do tiến trình khác ghi, chỉ báo trạng thái).

---

## 1. Kiểm tra đầy đủ so với danh sách yêu cầu

### 1.1 Kết quả tổng quát [Đã kiểm]

Toàn bộ **63/63** file trong danh sách `docs/` đều EXISTS. Không file nào chỉ có heading hoặc filler. File ngắn nhất `docs/design/UI_UX_GUIDELINES.md` = 149 dòng, `docs/product/NON_FUNCTIONAL_REQUIREMENTS.md` = 161 dòng; không file nào < 60 dòng. Ngoài danh sách còn có các file bổ sung (INFRASTRUCTURE_DESIGN, HERDR_DESIGN_INTEGRATION, USABILITY_AUDIT_CHECKLIST, USER_JOURNEY, UX_RESEARCH_REPORT, DEVSECOPS_GUIDE, CLAUDE_CODE_INTEGRATION, CODEX_INTEGRATION, HERDR_ANALYSIS, ORCHESTRATION_WORKFLOW_RESEARCH).

Kiểm tra cấu trúc dùng regex đa dạng (Mục đích/Mục tiêu/Purpose, Phạm vi/Scope/Trong:, Acceptance/Tiêu chí/checklist `- [ ]`/AC-xxx, Rủi ro/Risk/đánh đổi, Câu hỏi mở/[Cần người dùng phê duyệt]). Chỉ các file sau thiếu ít nhất một mục (còn lại đủ cả 5):

| File | Dòng | Thiếu | Đánh giá |
|---|---|---|---|
| product/NON_FUNCTIONAL_REQUIREMENTS.md | 161 | Acceptance | NFR có cột "cách đo" nhưng không có checklist; AC tương ứng nằm ở ACCEPTANCE_CRITERIA (NFR-0xx) — chấp nhận được |
| SECURITY.md | 576 | Acceptance (checklist) | Có chiến lược test §9 (L511) và milestone M1…; không có checklist nghiệm thu riêng → P2 |
| research/CHATDEV_ANALYSIS.md | 656 | Phạm vi, Acceptance, Rủi ro | Tài liệu phân tích, không phải đặc tả — chấp nhận |
| research/CLAUDE_CODE_INTEGRATION.md | 738 | Phạm vi, Rủi ro | Có checklist (7); thiếu khung Phạm vi/Rủi ro tường minh → P2 |
| research/MY_VIRTUAL_OFFICE_ANALYSIS.md | 483 | Acceptance, Rủi ro | Phân tích — chấp nhận (rủi ro AGPL nằm ở REUSE_DECISIONS RR-1) |
| research/OPENHANDS_ANALYSIS.md | 369 | Phạm vi, Acceptance | Phân tích — chấp nhận |
| research/PIXEL_AGENTS_ANALYSIS.md | 497 | Acceptance | Phân tích — chấp nhận |

Ghi chú: `DATABASE_DESIGN.md` (1472 dòng) có Mục đích/Phạm vi nhưng dưới tiêu đề khác ("Mục tiêu", "Trong/Ngoài"), nên không phải thiếu thật.

### 1.2 Bảng file / số dòng [Đã kiểm]

| Nhóm | File : dòng |
|---|---|
| product | PRD 236 · SRS 254 · PRODUCT_VISION 211 · FUNCTIONAL_REQUIREMENTS 323 · NON_FUNCTIONAL_REQUIREMENTS 161 · USER_ROLES_AND_PERMISSIONS 161 · USER_FLOWS 220 · ACCEPTANCE_CRITERIA 194 |
| docs/ gốc | SYSTEM_ARCHITECTURE 317 · DATABASE_DESIGN 1472 · API_DOCUMENTATION 451 · FRONTEND_ARCHITECTURE 238 · BACKEND_ARCHITECTURE 220 · AGENT_ARCHITECTURE 294 · MULTI_AGENT_ORCHESTRATION 341 · WORKFLOW_ENGINE 444 · PROVIDER_INTEGRATION 525 · VIRTUAL_OFFICE 387 · EVENT_SYSTEM 427 · SECURITY 576 · IMPLEMENTATION_ROADMAP 560 · PROJECT_RISKS 210 |
| architecture | SKILL_SYSTEM 697 · WORKSPACE_AND_GIT_ISOLATION 335 · AGENT_ORGANIZATION 216 · AGENT_SCHEDULER 279 · RESOURCE_MANAGEMENT 232 · AGENT_COLLABORATION 180 |
| design | UI_UX_GUIDELINES 149 · DESIGN_SYSTEM 212 · DESIGN_TOKENS 438 · COMPONENT_SPECIFICATION 490 · SCREEN_SPECIFICATION 714 · NAVIGATION_STRUCTURE 255 · RESPONSIVE_DESIGN 188 · INTERACTION_DESIGN 184 |
| security | THREAT_MODEL 210 · SANDBOX_DESIGN 344 · CREDENTIAL_MANAGEMENT 250 · PERMISSION_MODEL 356 · AUDIT_LOGGING 284 |
| research | CHATDEV 656 · PIXEL_AGENTS 497 · MY_VIRTUAL_OFFICE 483 · OPENHANDS 369 · LICENSE_AND_REUSE 405 · ARCHITECTURE_COMPARISON 252 · REUSE_DECISIONS 269 · TECHNOLOGY_EVALUATION 326 · MULTI_AGENT_SCALABILITY 187 |
| development | DEVELOPMENT_GUIDE 271 · TESTING 246 · DEPLOYMENT 214 · ENVIRONMENT_SETUP 301 · CI_CD 296 · CODING_STANDARDS 253 |
| operations | AGENT_OPERATIONS 202 |

### 1.3 `docs/adr/`
Thư mục tồn tại nhưng **rỗng (0 file)** tại thời điểm rà [Đã kiểm]. LICENSE_AND_REUSE.md:385 và REUSE_DECISIONS.md:265 đều yêu cầu một ADR (MVO = REF-ONLY clean-room) — hiện chưa có.

### 1.4 File gốc

| File | Trạng thái | Dòng | Nhận xét |
|---|---|---|---|
| PROJECT_STATUS.md | EXISTS | 18 | **Lỗi thời**: ghi "Phase 0 ĐANG CHẠY — 10 agent nghiên cứu song song, 1 agent UI/UX chờ slot", không phản ánh 70+ tài liệu đã có; chưa có mục chờ-duyệt/blocker |
| WORK_PLAN.md | EXISTS | 54 | Còn "Prisma/Drizzle", "vitest/jest" (chưa cập nhật theo D-002) |
| IMPLEMENTATION_LOG.md | EXISTS | 6 | Gần như trống |
| DECISION_LOG.md | EXISTS | 34 | D-001…D-020, tất cả "Chờ duyệt" (D-007 thiếu ô) |
| KNOWN_ISSUES.md | EXISTS | 6 | Gần như trống dù tài liệu đã nêu hàng chục mâu thuẫn/hạng mục chặn freeze |
| NEXT_SESSION.md | **MISSING** | — | Đang được tiến trình khác ghi (theo brief) |
| AGENT_WORK_LOG.md | EXISTS | 37 | — |
| TASK_BACKLOG.md | EXISTS | 45 | — |
| AGENTS.md | EXISTS | 56 | Tuân thủ chính sách credential (L27) |
| PHASE_0_FINAL_REPORT.md | **MISSING** | — | Đang được tiến trình khác ghi |

---

## 2. Nhất quán chéo giữa các tài liệu [Đã kiểm bằng grep]

Quy ước: "Chính tắc" = giá trị mà DECISION_LOG/tài liệu hoà giải chọn. "Lệch" = nơi còn giá trị khác chưa được vá.

| # | Chủ đề | Giá trị chính tắc | Nơi còn lệch (file:dòng) | Mức |
|---|---|---|---|---|
| 1 | **ORM** | Drizzle (D-002; DATABASE_DESIGN §1; SYSTEM_ARCHITECTURE:65,232; BACKEND_ARCHITECTURE) | `architecture/AGENT_ORGANIZATION.md:41,42,66,67,138` dùng `prisma/**` làm path sở hữu của `db_infra`/`backend` và skill `prisma`; `SECURITY.md:241,376,430,432` (`prisma migrate`, `schema.prisma`, `prisma/migrations`); `security/PERMISSION_MODEL.md:101,120`; `development/ENVIRONMENT_SETUP.md:178` ("Prisma/Drizzle/TypeORM"); `WORK_PLAN.md`; `research/ORCHESTRATION_WORKFLOW_RESEARCH.md` (Prisma schema, 9 lần); `research/MY_VIRTUAL_OFFICE_ANALYSIS.md:461`. SYSTEM_ARCHITECTURE:103,300 và MULTI_AGENT_ORCHESTRATION:325 tự ghi "Prisma chỉ là minh hoạ" | P1 (manifest ownership trỏ sai thư mục) |
| 2 | **Concurrency mặc định / tối đa** | mặc định 5, tối đa 18 (D-010; RESOURCE_MANAGEMENT:48,152; API_DOCUMENTATION:202; BACKEND:102; ORCH:74; AGENT_OPERATIONS:22) | `DATABASE_DESIGN.md:1315` seed `max_concurrent_agents = 3`; `provider_configs.max_concurrent` mặc định 2 (AGENT_ARCHITECTURE:280 M-07 chỉ nói "DB default chỉ là khởi tạo"); ORCH §2 từng đặt 10 (đã ghi M-O2/BACKEND:125 → theo 5); `RESOURCE_MANAGEMENT:49` provider mặc định claude 5 / codex 3 (A07 từng 6); `WORKSPACE_AND_GIT_ISOLATION:78,199` worktree 6/project, 12 toàn hệ thống; `INFRASTRUCTURE_DESIGN:151,178` sandbox 3, preview 3. "10" trong `DEVELOPMENT_GUIDE.md:98,145,236` và `IMPLEMENTATION_ROADMAP.md:33` là giới hạn subagent khi **phát triển** (không phải sản phẩm) — đúng, nhưng dễ gây hiểu nhầm | P1: seed DB 3 ≠ 5 chưa hoà giải |
| 3 | **Office engine** | Phaser 3.90 (D-011; VIRTUAL_OFFICE §4; FRONTEND_ARCHITECTURE) | `design/SCREEN_SPECIFICATION.md:142` ("PixiJS/Canvas 2D [Giả định]"), `:690` ("engine chọn (PixiJS/Canvas)"). Đã tự ghi nhận ở COMPONENT_SPECIFICATION:472 và ROADMAP:469 | P2 |
| 4 | **Tile size** | art 16 px, hiển thị zoom nguyên 2× = 32 px (VIRTUAL_OFFICE:60; PIXEL_AGENTS_ANALYSIS: TILE_SIZE=16 — đã đối chiếu mã, xem §3) | `DATABASE_DESIGN.md:1022` `tile_size` mặc định 32; SCREEN_SPECIFICATION §3.3 "32px tiles"; kích thước map DB 40×24 vs thiết kế 64×40 (VIRTUAL_OFFICE:116,215,362). VIRTUAL_OFFICE tự đề xuất đổi ngữ nghĩa cột | P1 (cần migration/đổi default trước khi build) |
| 5 | **agent_status** | 12 giá trị: created, idle, available, assigned, running, waiting, blocked, reviewing, completed, failed, paused, disabled — thống nhất PRD:57, FR-024, DATABASE_DESIGN:167, AGENT_ARCHITECTURE:55 | Phép chiếu sang event/office: `idle|working|waiting|error|offline` (SYSTEM_ARCHITECTURE:306) và `OfficeStatusCounts{working,waiting,blocked,failed,idle,offline}` (COMPONENT_SPECIFICATION:334) — không có trong 12 trạng thái; BACKEND_ARCHITECTURE:205 vẫn hỏi "cần xác nhận với Database agent". Thêm lớp 5 attention state (D-017) | P2 (bảng ánh xạ chưa chốt) |
| 6 | **Permission levels** | Chưa chốt: USER_ROLES §4.2 = L0–L5 (6 mức); AGENT_ORGANIZATION = L0..L4 (5 mức, theo AGENT_ARCHITECTURE:148); DB enum `permission_level` = read_only/standard/elevated/full (DATABASE_DESIGN:170, 4 mức) | PERMISSION_MODEL:36–48 đề xuất ánh xạ L0–L5 → 4 enum + `permission_profile`; PERMISSION_MODEL:321 nêu "`full` vô nghĩa"; DB chưa sửa | P1 |
| 7 | **Audit table** | `audit_logs` (DATABASE_DESIGN, AUDIT_LOGGING:44, EVENT_SYSTEM:353,372,426, BACKEND, API) | `audit_events` còn ở `SECURITY.md:484`, `architecture/INFRASTRUCTURE_DESIGN.md:185,212`, `development/DEPLOYMENT.md:125,143,163,182`, `operations/DEVSECOPS_GUIDE.md` (7 lần, L106,116,142,147,174,207); `architecture/SKILL_SYSTEM.md:497,645` dùng `audit_log` (số ít) | P1 |
| 8 | **Audit retention** | 365 ngày online + export JSONL.age ký HMAC thêm 2 năm (AUDIT_LOGGING:158,239; DEVSECOPS:116,170,207) | `SECURITY.md` §8 ghi 180 ngày (theo AUDIT_LOGGING:154,265); AUDIT_LOGGING đánh dấu "Cần người dùng phê duyệt". Còn hai con số trong hai tài liệu bảo mật | P1 |
| 9 | **Approval timeout** | Chưa thống nhất — năm giá trị: 24 h (RESOURCE_MANAGEMENT:104; AGENT_OPERATIONS:142 SLA 24h), 540 s hook HTTP (SECURITY.md:400), 10 phút mức tool (PROVIDER_INTEGRATION:411), 30 phút `canUseTool` (research CLAUDE_CODE_INTEGRATION:512), 3600 s `timeout_sec` DB (PERMISSION_MODEL:323), "không timeout" (WORKFLOW_ENGINE:201,359; USER_ROLES §5.5) | PROVIDER_INTEGRATION:411 và PERMISSION_MODEL §6.4 đề xuất hợp nhất theo tầng; chưa ghi vào DECISION_LOG | P1 |
| 10 | **Event naming** | UPPER_SNAKE (EVENT_SYSTEM:39,72,172,364) | Dotted lowercase ở `DATABASE_DESIGN.md:1196`, `API_DOCUMENTATION.md:343`, `security/AUDIT_LOGGING.md:24`; API_DOCUMENTATION:397,433,451 liệt kê là "hạng mục chặn freeze (chưa giải quyết)"; D-014 trích "Prompt 1 mục 8.4" mà API_DOCUMENTATION:433 xác nhận **không có trong docs/**. Cũng còn lệch `task_status` HOA vs enum khác thường (DATABASE_DESIGN:201; DECISION_LOG mâu thuẫn #5) | P1 |
| 11 | **Claude approval mechanism** | Hai tầng (PROVIDER_INTEGRATION:410): `canUseTool` = hỏi người; SDK in-process `PreToolUse` = policy cứng (D-004; research CLAUDE_CODE_INTEGRATION:503) | `SECURITY.md:119,135,351–354,551,565` thiết kế **PreToolUse hook kiểu HTTP** (`--settings`, `claude -p`) là "nguồn chính" — khác cơ chế SDK của D-004. PROVIDER_INTEGRATION:499 ghi "cần cập nhật SECURITY", chưa làm | P1 |
| 12 | **Test runner** | Vitest (TESTING.md ×15, FRONTEND, ROADMAP, VIRTUAL_OFFICE, TE-11) | `research/TECHNOLOGY_EVALUATION.md:243,310`: `apps/api` Vitest + `unplugin-swc` nếu spike qua, nếu không Jest (chưa chốt); `AGENT_ORGANIZATION.md:41` skill `jest`; `WORK_PLAN.md:47` "vitest/jest" | P2 |
| 13 | **Provider kind** | `claude_code | codex | openhands | generic_cli | mock` (PROVIDER_INTEGRATION:35) | `DATABASE_DESIGN.md:171–173` enum `provider_kind` = `claude_code, anthropic_api, openai, gemini, ollama, openrouter, custom` — **không có `codex`/`openhands`**; API_DOCUMENTATION:397 nêu "tên provider kind" là hạng mục chặn freeze | **P0** |
| 14 | **Biến môi trường chế độ mock** | — | `development/CI_CD.md:152,168,265` và `DEPLOYMENT.md:159` dùng `AGENT_MODE=mock`; `TESTING.md:94,242` và ROADMAP:442 dùng `AIW_PROVIDER_MODE=fake`; PROVIDER_INTEGRATION:35 dùng `AIW_TEST_MODE=ci` — ba tên khác nhau cho cùng một cơ chế; guard "từ chối khởi động ngoài NODE_ENV=test" chỉ gắn với tên của TESTING | P1 |
| 15 | **Đường dẫn tài liệu chéo** | — | Liên kết hỏng (đã đối chiếu bằng script): `docs/security/SECURITY.md` (thực tế `docs/SECURITY.md`) trong 6 file (INFRASTRUCTURE_DESIGN, WORKSPACE_AND_GIT_ISOLATION, CI_CD…); `docs/architecture/DATABASE_DESIGN.md` (thực tế `docs/DATABASE_DESIGN.md`) trong 4 file (INFRASTRUCTURE_DESIGN, WORKSPACE_AND_GIT_ISOLATION, DEPLOYMENT…); `docs/DECISION_LOG.md` (ARCHITECTURE_COMPARISON; thực tế ở gốc); `docs/security/VERIFICATION_LOG.md` (SANDBOX_DESIGN, file không tồn tại) | P2 |

### 2.1 Tuyên bố "Đã duyệt" [Đã kiểm]
`grep` "Đã duyệt|Approved|Owner đã duyệt|…" trên toàn bộ docs + file gốc: mọi lần xuất hiện đều là (a) checklist chưa tick ("- [ ] Owner đã duyệt các mục…": AGENT_ARCHITECTURE:294, EVENT_SYSTEM:427, MULTI_AGENT_ORCHESTRATION:341, WORKFLOW_ENGINE:444, UI_UX_GUIDELINES:149), (b) nhãn UI ("Đã duyệt" trong EVENT_SYSTEM:335), (c) quy tắc quy trình (AGENTS.md:9, ROADMAP P0-14 TODO). **Không có vi phạm.**
Quan sát kèm theo: `DECISION_LOG.md` D-007 thiếu giá trị cột "Phê duyệt" (hàng bị lệch cột) — không ai có thể đọc đó như "đã duyệt", nhưng cần sửa cho nhất quán.
Nhãn "[Đã xác minh]" bị dùng rộng hơn nghĩa "đã chạy/kiểm tra thực tế": ví dụ BACKEND_ARCHITECTURE:114 gắn "[Đã xác minh]" cho việc danh sách queue **khớp với tài liệu khác**, DEVELOPMENT_GUIDE:98,145,236 gắn cho giới hạn subagent của Orchestrator. Đây là xác minh nội bộ, không phải bằng chứng thực — xem P2-06.

---

## 3. Kiểm chứng các tuyên bố "Đã xác minh" (spot check)

Tài liệu research CLAUDE_CODE_INTEGRATION, CODEX_INTEGRATION, OPENHANDS_ANALYSIS, PIXEL_AGENTS_ANALYSIS, LICENSE_AND_REUSE **không dùng nhãn chữ "[Đã xác minh]"** (chỉ SKILL_SYSTEM dùng 21 lần); tôi kiểm các khẳng định thực tế có thể đối chiếu trong các tài liệu này.

| # | Tuyên bố (nguồn trong docs) | Cách kiểm | Kết quả |
|---|---|---|---|
| 1 | License ChatDev = Apache-2.0, "Copyright 2025 OpenBMB", không có NOTICE (LICENSE_AND_REUSE:17,34–35) | `head LICENSE`; `grep Copyright`; `ls NOTICE*` | **CONFIRMED** (LICENSE Apache 2.0; dòng 189 "Copyright 2025 OpenBMB"; không có NOTICE) |
| 2 | pixel-agents = MIT, "Copyright (c) 2026 Pablo De Lucca", không CREDITS/NOTICE; credit asset ở README dòng 48 (LICENSE_AND_REUSE:18) | đọc LICENSE, `ls`, `sed -n 48p README.md` | **CONFIRMED** (README:48 nêu JIK-A-4 Metro City) |
| 3 | my-virtual-office = AGPL-3.0; OpenHands (agent-canvas) = MIT "© 2025 OpenHands contributors", v1.34.0 (LICENSE_AND_REUSE:19–20; REUSE_DECISIONS:15,136) | đọc LICENSE + `package.json` | **CONFIRMED** (GNU AFFERO v3; MIT; `@openhands/agent-canvas` 1.34.0, `"license": "MIT"`). herdr = Apache-2.0 (`Cargo.toml` 0.9.5) cũng đúng |
| 4 | Commit id: ChatDev `a2ea4a2`, my-virtual-office `f875345` (2026-08-15, v0.7.4), OpenHands `7219c19` (2026-10-29), herdr `ee5bfb50` | `git -C <repo> log -1 --format=%h` | **CONFIRMED** cả 4 (a2ea4a2, f875345, 7219c19…, ee5bfb5…). Pixel Agents: HEAD là `b1bafd0`, `package.json` version 1.5.0 khớp "v1.5.0"; id `add749a` trong REUSE_DECISIONS:217 **[Không kiểm được]** |
| 5 | MVO: `server.py` 22.229 dòng, `game.js` 17.334, `chat.js` 5.506, `projects.js` 2.315, `style.css` 5.011 (MY_VIRTUAL_OFFICE_ANALYSIS:43–44) | `wc -l` | **CONFIRMED chính xác tuyệt đối** (22229/17334/5506/2315/5011) |
| 6 | Pixel Agents: `TILE_SIZE=16`, `MAX_COLS/ROWS=64`, default layout 21×22 + 36 furniture, 6 character sheet (PIXEL_AGENTS_ANALYSIS:15,16,278) | `grep` constants.ts; node đọc `default-layout-1.json`; `ls characters` | **CONFIRMED** (`webview-ui/src/constants.ts:4,7,8`; layout cols 21 rows 22 furniture 36; `char_0..5.png`) |
| 7 | Pixel Agents: `core/src/messages.ts:126` `AgentActivityStatus = 'active'|'waiting'`; `agentManager.ts:64-81` createTerminal; `makeHookEntry` dòng 536; script `~/.pixel-agents/hooks/claude-hook.js` (PIXEL_AGENTS_ANALYSIS:70,110,194) | `sed -n 126p`; `sed -n 64,67p`; `grep -n makeHookEntry` | **CONFIRMED cả 4** (messages.ts:126 đúng nguyên văn; agentManager.ts:64 createTerminal; claudeHookInstaller.ts:536; comment :39 nêu đường dẫn hook) — tài liệu không ghi path đầy đủ cho `makeHookEntry` nhưng dòng đúng |
| 8 | SKILL_SYSTEM §3.5: ChatDev `.agents/skills/<name>/SKILL.md`, `runtime/node/agent/skills/manager.py` với `activate_skill`, `<available_skills>`, `entity/configs/node/skills.py` | `ls`, `grep` | **CONFIRMED** (3 skill: greeting-demo, python-scratchpad, rest-api-caller; manager.py có `build_available_skills_xml` :149 và `activate_skill` :173; skills.py tồn tại). `read_skill_file` chưa grep **[Không kiểm được]** |
| 9 | SKILL_SYSTEM §3.4/OPENHANDS_ANALYSIS: `src/utils/skill-scope.ts` markers `/.agents/skills/`, `/.openhands/skills/`, `/.openhands/microagents/`; `skills-service.api.ts` gọi `/api/v1/skills/search` và `/api/v1/app-conversations/{id}/skills`; `scripts/ingress.mjs` route 18001/18000 | `grep` | **CONFIRMED** (skill-scope.ts:16–18; skills-service.api.ts:41,79; ingress.mjs:10,129–131) |
| 10 | SKILL_SYSTEM §3.6: pixel-agents không có khái niệm skill | `grep -rli skill` trên .ts/.md | Không có kết quả → **CONFIRMED** |
| 11 | Claude CLI **2.1.312** (CLAUDE_CODE_INTEGRATION:6) | `claude --version` | **CONFIRMED** "2.1.312 (Claude Code)" |
| 12 | `claude auth status` trả JSON có `loggedIn`, `authMethod: "claude.ai"`, `subscriptionType: "team"` (CLAUDE_CODE_INTEGRATION:23) | `claude auth status` (đã che email/org khi hiển thị; không in credential) | **CONFIRMED** (loggedIn true; authMethod claude.ai; apiProvider firstParty; subscriptionType team) |
| 13 | Flag có trong `--help`: `--add-dir`, `--bare`, `--settings`, `--session-id`, `--resume`, `--permission-mode`; `--permission-prompt-tool` **không còn là flag riêng** (CLAUDE_CODE_INTEGRATION:89) | `claude --help` + grep | **CONFIRMED** (các flag có mặt; `--permission-prompt-tool` chỉ xuất hiện trong mô tả của flag khác, không có dòng flag riêng) |
| 14 | Codex CLI chưa cài global; bundled `codex.exe` = `codex-cli 0.164.0-alpha.11.1`; `~/.codex/auth.json` tồn tại; npm latest `@openai/codex` 0.166.0 (CODEX_INTEGRATION:13–21) | `where codex`; chạy `codex.exe --version`; `ls ~/.codex`; `npm view` | **CONFIRMED** cả 4 (`where` không tìm thấy; version khớp; thư mục có `auth.json`, `config.toml`; 0.166.0). Chỉ liệt kê tên, không đọc nội dung |
| 15 | Máy tham chiếu: i5-11400H, 16 GB, 6C/12T, ổ D còn ~109 GB, pnpm 10.34.6 (RESOURCE_MANAGEMENT:182; ROADMAP:520; ENVIRONMENT_SETUP:34; CODEX_INTEGRATION:13) | PowerShell CIM; `pnpm --version` | **CONFIRMED** (i5-11400H, 12 logic, RAM 15.8 GB, D: trống 108.2 GB, pnpm 10.34.6; free RAM hiện 4.1 GB — "6.4 GB free" là số tại thời điểm đo trước, hợp lý) |
| 16 | `.credentials.json` "617 bytes, mode 644" (CLAUDE_CODE_INTEGRATION:158; SECURITY.md:35,533; THREAT_MODEL:24; CREDENTIAL_MANAGEMENT:20) | chỉ `ls -l` (metadata, không đọc nội dung) | **LỆCH/không chính xác**: hiện **551 bytes** (token refresh làm đổi kích thước — không phải lỗi gốc, nhưng không nên coi 617 là hằng số). "mode 644" là artifact của Git Bash/MSYS trên NTFS; Windows dùng ACL — rủi ro R2 ("mọi process cùng user đọc được") vẫn đúng về bản chất nhưng nên diễn đạt theo ACL, không theo `chmod` |
| 17 | Phaser 3.90 (D-011), Drizzle "v1.x" (DATABASE_DESIGN:29), `@anthropic-ai/claude-agent-sdk`, `@openhands/typescript-client` | `npm view` | `phaser@3.90.0` **tồn tại** nhưng `latest` = 4.2.1 (D-011 chỉ nói "cân nhắc Phaser 4 sau" — nên ghi lý do ở lại 3.x); **`drizzle-orm` latest ổn định = 0.45.4; v1.0 mới chỉ có tag beta** → nhãn "Drizzle v1.x" **không chính xác** (xem P1-08); SDK Claude 0.3.314 và `@openhands/typescript-client` 1.59.0 **tồn tại** |

Tổng: 17 hạng mục kiểm, **15 CONFIRMED hoàn toàn, 1 LỆCH nhỏ (#16), 1 PARTIAL (#17: đúng sự tồn tại, sai mô tả trạng thái phiên bản Drizzle)**. Không phát hiện tuyên bố bịa đặt về mã nguồn các repo tham chiếu.

---

## 4. Tuân thủ chính sách [Đã kiểm bằng grep]

| Chính sách | Kết quả | Chi tiết |
|---|---|---|
| Không đọc `~/.claude/.credentials.json` / `~/.codex/auth.json` | **ĐẠT** | Mọi lần xuất hiện (≈50) là cấm/giải thích/test cấm (AGENTS.md:27, SECURITY.md:22,285, CODING_STANDARDS F-05, TESTING:144 Semgrep, PROVIDER_INTEGRATION:321,419,518, CREDENTIAL_MANAGEMENT:20,43). Không có hướng dẫn đọc. Hai ca cần lưu ý: (1) `PROVIDER_INTEGRATION.md:263` — OpenHands ACP sub-mode "materialize `CODEX_AUTH_JSON` → `~/.codex/auth.json`" mâu thuẫn D-008, **đã tự tắt mặc định** (chỉ MVP native + key trong keychain); (2) `SANDBOX_DESIGN.md:205` Tier 2 dùng `ANTHROPIC_API_KEY` riêng qua Docker secret tmpfs — đúng hướng, nhưng đổi mô hình tính phí (SECURITY.md:191 đã nêu) |
| Không lưu token plaintext | **ĐẠT** | Chỉ xuất hiện như điều cấm (CODING_STANDARDS F-04/F-06, AC-113). `localStorage` được dùng cho `currentProjectId`, density, draft form — không phải secret. `OPENHANDS_ANALYSIS:65` mô tả `apiKey` lưu localStorage **của OpenHands** (tham khảo, không phải thiết kế của ta) |
| Không mock/fake trong luồng sản phẩm | **ĐẠT CÓ ĐIỀU KIỆN** | CODING_STANDARDS F-03 cấm; PROVIDER_INTEGRATION:35,446 đặt `MockProviderAdapter` ở `packages/provider-testkit`, `kind:'mock'` chỉ tồn tại khi chế độ test; TESTING:94 guard từ chối khởi động. Điểm cần sửa: (a) `ProviderKind` trong **contract production** vẫn chứa literal `'mock'` (PROVIDER_INTEGRATION:35) → nên nằm trong type riêng của testkit; (b) `DEPLOYMENT.md:159` dùng "tạo task mock (`AGENT_MODE=mock`)" trong checklist **smoke test deploy** — trộn mock vào luồng release (P1-09); (c) `API_DOCUMENTATION.md:380,391` dùng MSW mock cho FE ở giai đoạn phát triển rồi "thay bằng API thật" — chấp nhận nếu bị chặn khỏi bundle (ROADMAP P1-63 có kiểm bundle) |
| Không hardcode 12/18 agent trong thiết kế mức code | **ĐẠT** | D-016, CODING_STANDARDS F-01 cấm; AGENT_ORGANIZATION:22 ("role là dữ liệu"); mọi `18` trong code-level là trần cấu hình (`z.number().max(18).default(5)` RESOURCE_MANAGEMENT:152) hoặc tham số benchmark (T0–T7). 12 trạng thái lifecycle là enum DB (có chủ đích). Ngoại lệ nhẹ: SCREEN_SPECIFICATION:320 hành vi UI cứng "đạt 18 agent → cảnh báo 'Giới hạn 18'" nên đọc từ cấu hình thay vì hằng 18 (P2-07) |
| Tính năng thiếu Acceptance Criteria | **KHÔNG ĐẠT** | Xem P0-1: 50/199 FR không xuất hiện trong ACCEPTANCE_CRITERIA, trong đó 31 là Must/MVP; 30 số AC bị bỏ trống trong dải 001–130 mà TESTING.md/ROADMAP vẫn tham chiếu |
| Không auto-deploy production | **ĐẠT** | DEPLOYMENT.md §9 ("không bao giờ tự động deploy"), FR-126/FR-199, WORKFLOW_ENGINE V12 ép gate `mandatory` cho `target=production`, template không chứa production (WE-09). Căng thẳng nhỏ: DEPLOYMENT §9 phát biểu tuyệt đối "không bao giờ deploy production" trong khi WORKFLOW_ENGINE cho phép node production **có approval** — nên thống nhất câu chữ (P2-08). `onTimeout=auto_approve` tồn tại nhưng bị cấm cho gate `mandatory` (WORKFLOW_ENGINE:100,232) |
| AGPL (my-virtual-office) — không copy mã/prompt/layout | **ĐẠT** | LICENSE_AND_REUSE, REUSE_DECISIONS §4.4 (DENY RD-060, clean-room, cấm đường dẫn `my-virtual-office` trong repo), VIRTUAL_OFFICE:222 từ chối import `.mvo-layout.json`, SCREEN_SPECIFICATION:280 không dùng asset repo tham chiếu. Mọi mục "Có" liên quan MVO là ý tưởng (REF-ONLY). Rủi ro quy trình: RR-1 (dev đọc mã MVO rồi viết lại) — mới ở mức quy định, ADR chưa có (P1-10) |
| Ngoài: herdr / Apache-2.0, ChatDev Apache-2.0 | ĐẠT | Yêu cầu attribution + change notice đã nêu (REUSE_DECISIONS:217–219). `THIRD_PARTY_NOTICES.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, `README.md` được tham chiếu nhưng chưa tồn tại (là sản phẩm Phase 1; chấp nhận) |

---

## 5. Mối lo ngại về tính khả thi (xếp hạng, tối đa 10)

1. **Cơ chế phê duyệt Claude trên Windows (SDK `canUseTool` + hook) chưa kiểm chứng đầu-cuối, và hai tài liệu mô tả hai cơ chế khác nhau.** research CLAUDE_CODE_INTEGRATION có smoke test SDK in-process (canUseTool chặn được Write — mức "đã chạy 1 lần"), nhưng SECURITY.md dựa vào **HTTP PreToolUse hook** (`"type":"http"`, timeout 600 s, giữ kết nối HTTP mở chờ người) — SECURITY.md:566 tự ghi hook type `http` chạy theo Git Bash→PowerShell trên Windows; chưa có bằng chứng chạy thật. Hook HTTP giữ kết nối tối đa 540 s mâu thuẫn với "approval có thể chờ giờ/ngày". [Không kiểm được hook `http`; xem P1-05]
2. **18 session đồng thời vs hạn mức subscription dùng chung (1 tài khoản `team`) và điều khoản dịch vụ.** Tài liệu thừa nhận: giới hạn thật là quota, không phải process (RESOURCE_MANAGEMENT:112; D-010; DECISION_LOG mâu thuẫn #2 "cần xác minh điều khoản"). Chỉ có số đo 3 process và "5 song song" (PROVIDER_INTEGRATION:457); T7 18-concurrency chưa chạy. Thiết kế đúng khi đặt mặc định 5; tiêu chí Phase 7 là chứng minh thật. Rủi ro: spawn tax 48k token × 18 (RESOURCE_MANAGEMENT:212) có thể làm AC-064 pass nhưng giá trị thực bằng 0.
3. **Bộ nhớ 16 GB cho 18 CLI + Docker Desktop + Postgres + Redis + Playwright.** RESOURCE_MANAGEMENT:188 tự kết luận "nặng: không trên 16 GB"; free RAM hiện 4.1 GB (tôi đo). Docker Desktop VM thường chiếm 2–6 GB; `INFRASTRUCTURE_DESIGN.md:151` khuyến nghị `.wslconfig memory=12GB` **cho máy 32 GB** — áp vào máy thực 15.8 GB sẽ chiếm 76% RAM. Cần số liệu theo máy thực.
4. **Ranh giới bảo mật Tier 1 (CLI chạy trên host).** Yêu cầu owner "không chạy code AI trực tiếp trên host" mâu thuẫn với D-007 (DECISION_LOG mâu thuẫn #1); chính SECURITY.md R1/R2/R5 thừa nhận chính sách CLI "không phải ranh giới bảo mật", token 644/ACL đọc được bởi mọi process cùng user, Docker Desktop VM là root-equivalent. Docker daemon chưa chạy lúc khảo sát (DECISION_LOG #3) nên cgroup/ECI/rootless chưa xác minh (tôi chỉ xác nhận Docker CLI 29.8.1 có mặt, không kiểm daemon).
5. **Phaser + React.** Thiết kế ranh giới tốt (VIRTUAL_OFFICE §4: `office-core` thuần TS, Phaser.Game tạo một lần, bus typed, chịu StrictMode double-mount). Rủi ro còn lại: Phaser 3.90 trong khi `latest` là 4.2.1 (D-011 chỉ nói "cân nhắc sau" — nên ghi lý do/độ trễ migrate), headless test, 18+8 helper ≤16.6 ms/khung chỉ là mục tiêu chưa đo, tile 16 vs 32 (§2 #4).
6. **Drizzle + bảng phân vùng + FK vòng bằng SQL tay.** DATABASE_DESIGN §4.14/§12: `agent_events`, `audit_logs`, `usage_records` phân vùng theo tháng phải viết `CREATE TABLE ... PARTITION BY` thủ công (migration `0004` custom), drizzle-kit không biết bảng phân vùng → nguy cơ `drizzle-kit generate` sinh lại DDL xung đột, drift schema/TS type; FK vòng (`departments.head_agent_id`, `agents.current_task_id`) thêm tay. Thêm: Drizzle ổn định là 0.45.x, bản 1.x còn beta (tôi kiểm npm) → quyết định ORM đang dựa trên bản beta hoặc nhãn sai. Mức khả thi: cao nhưng cần spike migration/partition-maintainer trước khi freeze (roadmap chưa có task spike riêng cho việc này).
7. **44 bảng + event store + outbox + Redis Stream + BullMQ + Redis Lua semaphore + durable workflow tự xây (D-003).** Khối lượng cho một nhóm nhỏ rất lớn; ROADMAP đã nêu R-ENGINE-COMPLEXITY. Không phải "không thể xây" nhưng ước lượng Phase 3 (5 task thật, kill/resume) nhiều khả năng thấp; chưa có spike nào chứng minh crash recovery 20/20 (AC-127).
8. **Codex integration hoàn toàn chưa chạy.** Codex CLI chưa cài toàn cục (tôi xác nhận); `codex app-server` JSON-RPC, `approvals$`, `CODEX_HOME` riêng đều là [Giả định]. D-005 ưu tiên app-server (giao thức "experimental"). Rủi ro API thay đổi giữa 0.164→0.166.
9. **Skill/memory cho 2 CLI khác cơ chế nạp (`--add-dir` vs junction `.agents/skills` trên Windows).** SKILL_SYSTEM đã nêu junction `mklink /J` không cần admin — hợp lý, nhưng chưa có thử nghiệm Codex discover qua junction vì Codex CLI chưa cài; Claude `--add-dir` + `--bare` discover đã có docs nhưng tôi không chạy prompt để kiểm.
10. **Tauri 2 sidecar Node + Windows path có khoảng trắng + AV.** DEPLOYMENT.md hoãn Tauri sang sau MVP (hợp lý); rủi ro đường dẫn `Agent management` (có khoảng trắng) được nêu trong DATABASE_DESIGN nhưng chưa có test cho `aiw-exec`/hook script quoting trên PowerShell — thường là nguồn lỗi thực tế đầu tiên.

---

## 6. Verdict theo nhóm tài liệu

| Nhóm | Verdict | Lý do ngắn |
|---|---|---|
| docs/product (PRD, SRS, VISION, FR, NFR, USER_ROLES, USER_FLOWS, AC) | **PASS WITH FIXES** | Nội dung đủ/sâu; FR-024 12 state khớp DB. P0-1 (AC coverage + numbering), USER_ROLES L0–L5 chưa khớp DB |
| docs/ kiến trúc lõi (SYSTEM, BACKEND, FRONTEND, API, AGENT, ORCHESTRATION, WORKFLOW, EVENT, PROVIDER, VIRTUAL_OFFICE, ROADMAP, RISKS) | **PASS WITH FIXES** | Tự ghi nhận mâu thuẫn tốt; còn event naming, provider kind, tile size, approval timeout chưa chốt; ROADMAP có exit criteria thực chứng |
| DATABASE_DESIGN | **PASS WITH FIXES** | Rất chi tiết; cần sửa enum `provider_kind` (P0-2), seed concurrency 3, `tile_size`, nhãn Drizzle v1.x |
| docs/architecture (SKILL, WORKSPACE, ORGANIZATION, SCHEDULER, RESOURCE, COLLABORATION, INFRASTRUCTURE) | **PASS WITH FIXES** | ORGANIZATION còn `prisma/**`; INFRASTRUCTURE `audit_events` + .wslconfig 32 GB |
| docs/design (UI_UX, DESIGN_SYSTEM, TOKENS, COMPONENT, SCREEN, NAVIGATION, RESPONSIVE, INTERACTION, + HERDR, USER_JOURNEY, UX_RESEARCH) | **PASS** (kèm 2 sửa nhỏ P2) | SCREEN_SPECIFICATION còn "PixiJS/Canvas" `[Giả định]` và có 22 khối AC theo màn hình; chưa gắn ID AC chung |
| docs/security (THREAT_MODEL, SANDBOX, CREDENTIAL, PERMISSION, AUDIT_LOGGING) + SECURITY.md + DEVSECOPS | **PASS WITH FIXES** | Chất lượng cao, tuân thủ credential; nhưng SECURITY.md/DEVSECOPS chưa đồng bộ với AUDIT_LOGGING/PROVIDER_INTEGRATION (audit table, retention, cơ chế approval) |
| docs/research (9 bắt buộc + CLAUDE/CODEX/HERDR/ORCHESTRATION) | **PASS** | 15/15 claim kiểm lại khớp repo thực; sai lệch nhỏ #16/#17 |
| docs/development (GUIDE, TESTING, DEPLOYMENT, ENV_SETUP, CI_CD, CODING_STANDARDS) | **PASS WITH FIXES** | TESTING nêu AC-001…130 nhưng chỉ có 100 AC; 3 tên biến mock; DEPLOYMENT smoke dùng mock |
| docs/operations (AGENT_OPERATIONS, DEVSECOPS_GUIDE) | **PASS WITH FIXES** | Playbook tốt; `audit_events` + SLA 24h approval chưa thống nhất |
| docs/adr | **FAIL (tạm thời)** | Thư mục rỗng; ADR clean-room MVO được 2 tài liệu yêu cầu. Kiểm lại khi tiến trình khác ghi xong |
| File gốc (PROJECT_STATUS, WORK_PLAN, IMPLEMENTATION_LOG, DECISION_LOG, KNOWN_ISSUES, TASK_BACKLOG, AGENT_WORK_LOG, AGENTS) | **PASS WITH FIXES** | PROJECT_STATUS lỗi thời; KNOWN_ISSUES/IMPLEMENTATION_LOG gần trống; DECISION_LOG sai định dạng D-007 |
| NEXT_SESSION.md, PHASE_0_FINAL_REPORT.md | **CHƯA ĐÁNH GIÁ (MISSING)** | Chỉ báo trạng thái theo brief |

### 6.1 Danh sách sửa ưu tiên

**P0 (chặn Phase 0 sign-off / contract freeze P1-04)**

| ID | File | Vấn đề | Đề xuất sửa |
|---|---|---|---|
| P0-1 | `docs/product/ACCEPTANCE_CRITERIA.md`, `docs/development/TESTING.md:11`, `docs/IMPLEMENTATION_ROADMAP.md` | (a) 50/199 FR không xuất hiện trong ACCEPTANCE_CRITERIA, gồm 31 FR Must/MVP: FR-002,005,006,009,010,011 (company/department), FR-181,182,185 + 183,184,186,187 (dashboards), FR-199 (policy mặc định), FR-233 (18 role seed), FR-236/238 (message bus/monitoring), FR-241 (diff) và toàn bộ FR-261…284 (22 màn hình Premium UI; AC chỉ có dạng "AC:" cục bộ trong SCREEN_SPECIFICATION, không có ID/trace) ; (b) dải AC-001…130 có **30 ID chưa định nghĩa**: 016–020, 035–040, 065–070, 083–085, 097–100, 115–120 (file định nghĩa đúng 100 AC); TESTING.md:11 "AC-001…AC-130" và ROADMAP dùng các dải bao gồm ID không tồn tại; (c) bảng §4 "Tổng 100" khớp số dòng nhưng không khớp dải ID đã công bố | Thêm AC cho các FR Must/MVP chưa phủ (hoặc ghi rõ lý do hoãn trong bảng truy vết), gán ID AC cho từng màn hình SCREEN_SPECIFICATION, đánh số lại hoặc ghi "dành riêng/không dùng" cho 30 ID trống, sửa TESTING.md:11 và các dải trong ROADMAP; thêm bảng FR→AC đầy đủ (script đối chiếu: so FR ID trong FUNCTIONAL_REQUIREMENTS với cột FR của AC) |
| P0-2 | `docs/DATABASE_DESIGN.md:171–173`, `docs/PROVIDER_INTEGRATION.md:35` | Enum `provider_kind` trong DB (`claude_code, anthropic_api, openai, gemini, ollama, openrouter, custom`) **không chứa `codex` và `openhands`**, hai provider là mục tiêu MVP (D-005, D-006); contract dùng `claude_code|codex|openhands|generic_cli|mock`. Migration `0000_enums_core` sẽ khoá sai giá trị | Thống nhất một danh sách duy nhất trong `packages/contracts`; DB: `claude_code, codex, openhands, generic_cli` (+ API providers nếu giữ), bỏ `mock` khỏi enum production; ghi quyết định vào DECISION_LOG |

**P1 (sửa trước khi Phase 1 bắt đầu viết mã)**

| ID | File | Vấn đề | Đề xuất sửa |
|---|---|---|---|
| P1-01 | `docs/SECURITY.md:484`, `INFRASTRUCTURE_DESIGN.md:185,212`, `development/DEPLOYMENT.md:125,143,163,182`, `operations/DEVSECOPS_GUIDE.md`, `SKILL_SYSTEM.md:497,645` | Còn `audit_events`/`audit_log`; chính tắc là `audit_logs` (AUDIT_LOGGING:44) | Thay đồng loạt hoặc thêm chú thích ở đầu mục audit trỏ tới AUDIT_LOGGING |
| P1-02 | `SECURITY.md §8`, `DEVSECOPS_GUIDE.md:116`, `AUDIT_LOGGING.md:158` | Retention 180 vs 365 ngày (+2 năm export) | Chốt 365 trong DECISION_LOG khi owner duyệt; cập nhật SECURITY.md |
| P1-03 | RESOURCE_MANAGEMENT:104; SECURITY:400; PROVIDER_INTEGRATION:411; WORKFLOW_ENGINE:201; PERMISSION_MODEL:323 | 5 giá trị approval timeout | Ghi bảng "tool-level 10 phút / workflow-level policy / không auto-approve" vào DECISION_LOG làm một quyết định |
| P1-04 | `DATABASE_DESIGN.md:1315`, `provider_configs.max_concurrent` | Seed `max_concurrent_agents=3`, provider default 2 ≠ D-010 (5) | Đồng bộ seed với `ResourceConfig` (5; claude 5, codex 3) |
| P1-05 | `SECURITY.md:119,135,351–354,551,565` vs D-004 | Thiết kế HTTP PreToolUse hook (CLI `-p`) ≠ SDK in-process `canUseTool` + hook; PROVIDER_INTEGRATION:499 đã đòi cập nhật | Viết lại §5 SECURITY theo hai tầng PROVIDER_INTEGRATION §17; nếu giữ HTTP hook, chạy PoC Windows và ghi bằng chứng |
| P1-06 | `architecture/AGENT_ORGANIZATION.md:41,42,66,67,138` | Manifest ownership dùng `prisma/**`; repo Drizzle (`apps/api/src/database/**`, `drizzle/**`) → enforcement diff-gate sẽ không bảo vệ đúng thư mục | Thay `prisma/**` bằng đường dẫn Drizzle thực; đổi skill `prisma`→`drizzle`; cập nhật SECURITY:430–432 và PERMISSION_MODEL:101,120 |
| P1-07 | DATABASE_DESIGN:1022; SCREEN_SPECIFICATION §3.3; VIRTUAL_OFFICE:60,215 | tile 32 (DB/UI) vs 16 (art) và map 40×24 vs 64×40 | Chốt `tile_size=16` + `display_scale`, default 64×40; đổi trong DB trước migration |
| P1-08 | `DATABASE_DESIGN.md:29`; TECHNOLOGY_EVALUATION | Nhãn "Drizzle v1.x" nhưng npm `latest` = 0.45.4 (1.0 chỉ beta) | Ghi phiên bản thực (0.45.x hoặc 1.0 beta) và chính sách pin; thêm spike partition/migration |
| P1-09 | CI_CD:152,168,265; DEPLOYMENT:159; TESTING:94; PROVIDER_INTEGRATION:35 | 3 tên biến mock (`AGENT_MODE`, `AIW_PROVIDER_MODE`, `AIW_TEST_MODE`); `'mock'` nằm trong `ProviderKind` production; smoke test deploy dùng mock | Chọn một tên (`AIW_PROVIDER_MODE=fake` + guard `NODE_ENV=test`); chuyển `'mock'` sang type testkit; bỏ mock khỏi smoke release |
| P1-10 | `docs/adr/` | Rỗng; yêu cầu ADR clean-room MVO (LICENSE_AND_REUSE:385; REUSE_DECISIONS:265) | Ghi ADR-001 MVO REF-ONLY + clean-room (kèm checklist reviewer, lint chặn path `my-virtual-office`) |
| P1-11 | `PROJECT_STATUS.md`, `KNOWN_ISSUES.md`, `IMPLEMENTATION_LOG.md`, `NEXT_SESSION.md`, `PHASE_0_FINAL_REPORT.md` | Trạng thái lỗi thời/trống/thiếu | Cập nhật sau khi tiến trình đang ghi hoàn tất; KNOWN_ISSUES nên chép các hạng mục "chặn freeze" từ API_DOCUMENTATION:397 và các bảng "Mâu thuẫn" |
| P1-12 | `INFRASTRUCTURE_DESIGN.md:151` | Khuyến nghị `.wslconfig memory=12GB, processors=6` "trên máy 32 GB"; máy thực 15.8 GB | Đặt giá trị theo máy tham chiếu (vd. 6–8 GB) hoặc tham số hoá theo RAM |
| P1-13 | `API_DOCUMENTATION.md:433`, `DECISION_LOG.md` D-014 | D-014 trích "Prompt 1 mục 8.4" không có trong docs; event naming vẫn chưa chốt (UPPER_SNAKE vs dotted) | Chép đặc tả event của owner vào `docs/` (hoặc ghi nguồn); chốt UPPER_SNAKE + quy đổi dotted cho DB/audit trong DECISION_LOG |
| P1-14 | `product/USER_ROLES_AND_PERMISSIONS.md`, `DATABASE_DESIGN.md:170`, `security/PERMISSION_MODEL.md:36–48` | 3 mô hình mức quyền (L0–L5, L0..L4, 4 enum) | Duyệt ánh xạ PERMISSION_MODEL và cập nhật enum + `permission_profile` |

**P2 (vệ sinh/độ rõ)**

| ID | File | Vấn đề | Đề xuất sửa |
|---|---|---|---|
| P2-01 | `SCREEN_SPECIFICATION.md:142,690` | "PixiJS/Canvas [Giả định]" cũ | Đổi thành Phaser 3.90 (D-011, chờ duyệt) |
| P2-02 | `WORK_PLAN.md`, `ENVIRONMENT_SETUP.md:178`, `AGENT_ORGANIZATION.md:41` | Còn "Prisma/Drizzle", "vitest/jest" | Cập nhật theo D-002/TE-11 |
| P2-03 | Liên kết hỏng (§2 #15) | `docs/security/SECURITY.md`, `docs/architecture/DATABASE_DESIGN.md`, `docs/DECISION_LOG.md`, `docs/security/VERIFICATION_LOG.md` | Sửa đường dẫn đúng |
| P2-04 | `DECISION_LOG.md` D-007 | Thiếu cột "Phê duyệt" | Thêm "Chờ duyệt" |
| P2-05 | `SECURITY.md`, `CLAUDE_CODE_INTEGRATION.md:158`, `THREAT_MODEL.md:24`, `CREDENTIAL_MANAGEMENT.md:20` | "617 byte, mode 644" (thực tế 551 byte; Windows dùng ACL) | Bỏ số byte; diễn đạt "file người dùng đọc được bởi tiến trình cùng user (ACL)" |
| P2-06 | BACKEND_ARCHITECTURE:114; DEVELOPMENT_GUIDE:98,145,236 | Nhãn "[Đã xác minh]" cho đối chiếu nội bộ | Dùng nhãn "[Nhất quán với X]" hoặc chỉ dùng "Đã xác minh" khi có lệnh/bằng chứng |
| P2-07 | SCREEN_SPECIFICATION:320 | UI cứng "18 agent" | Đọc từ `max_concurrent_agents`/cấu hình |
| P2-08 | DEPLOYMENT §9 vs WORKFLOW_ENGINE V12 | "Không bao giờ deploy production" vs "có thể, cần approval" | Thống nhất câu chữ: "không bao giờ **tự động**; chỉ khi có approval mandatory" |
| P2-09 | D-011 | Phaser 3.90 khi Phaser `latest` là 4.2.1 | Ghi lý do giữ 3.x và điều kiện nâng cấp |

---

## 7. Những gì tôi KHÔNG kiểm chứng (trung thực)

- **Không chạy prompt nào** với Claude/Codex; không kiểm `canUseTool`, hook `type:"http"`, `--add-dir` discovery, `total_cost_usd` cumulative, `rate_limit_event`, 3–5 process song song, số đo RAM 230 MB/process, 8 172 ms. Tất cả các số đo/live test do tài liệu tự báo cáo đều **chưa được tái hiện**.
- **Không kiểm Codex app-server/JSON-RPC** (CLI chưa cài); chỉ xác nhận binary bundled, phiên bản, `where`, và sự tồn tại `auth.json` (không đọc nội dung).
- **Không kiểm daemon Docker** (chỉ thấy Docker CLI 29.8.1), nên cgroup/ECI/rootless/WSL `.wslconfig` chưa kiểm.
- **Không kiểm các tuyên bố từ web** (docs.openhands.dev, agentskills.io, code.claude.com/docs/hooks, thông số Postgres/Redis/BullMQ, giới hạn quota/điều khoản dịch vụ Anthropic/OpenAI), đặc biệt: hook `http` timeout 600 s, 1 536 ký tự mô tả skill, `project_doc_max_bytes` 32 KiB, `claude auth status` JSON schema ngoài các trường tôi thấy.
- **Chỉ spot-check ~17 nhóm tuyên bố**; không kiểm toàn bộ các dòng "Đã xác minh" (SKILL_SYSTEM ~21 lần, tôi kiểm §3.4–3.6 và cấu trúc repo ChatDev/OpenHands; §3.1–3.3, 4.3.x chưa đối chiếu nguồn web). Các số dòng khác của MVO (server.py:15098–15495, v.v.), `add749a` của Pixel Agents, `read_skill_file` của ChatDev chưa kiểm.
- **Không kiểm bằng ngữ nghĩa từng FR/AC**: phủ FR→AC dựa trên script regex tìm "FR-xxx" (hỗ trợ dải `FR-a…b` và danh sách `, nnn`); AC không ghi FR theo cách khác có thể bị tính thiếu (nhưng đã đối chiếu thủ công với các mẫu FR-261, FR-199, FR-241). Phép đối chiếu AC-xxx giữa các tài liệu cũng dựa trên regex.
- **Không đánh giá chất lượng thiết kế UI** (token màu, WCAG contrast, bố cục), SQL cụ thể (cú pháp Drizzle chưa biên dịch), hay số liệu chi phí/quota.
- **Không đánh giá pháp lý** (verdict AGPL/Apache chỉ kiểm văn bản LICENSE và các quy tắc nội bộ; tài liệu tự khuyến nghị luật sư rà soát).
- **Trạng thái tại thời điểm rà**: `docs/adr/`, `NEXT_SESSION.md`, `PHASE_0_FINAL_REPORT.md` có thể đã thay đổi sau khi tôi kiểm; các tiến trình khác có thể còn đang ghi tài liệu — các kết quả đếm dòng có thể lệch nhẹ so với thời điểm đọc.
