# AGENT_ORGANIZATION.md — Tổ chức Agent, Agent Registry và mô hình Identity

Tác giả: Agent 13 (Multi-Agent Scalability & Scheduling Architect) — Phase 0, chỉ tài liệu.
Ngày: 2026-10-30. Trạng thái: DRAFT — chờ owner phê duyệt.
Liên quan: `AGENT_SCHEDULER.md`, `RESOURCE_MANAGEMENT.md`, `AGENT_COLLABORATION.md`, `docs/research/MULTI_AGENT_SCALABILITY.md`, `docs/operations/AGENT_OPERATIONS.md`. Workflow engine: xem `ORCHESTRATION_WORKFLOW_RESEARCH.md` (A07) — tài liệu này KHÔNG thiết kế lại workflow engine.

Ký hiệu trạng thái từng mục: **[Đã xác minh]** đo/kiểm tra thực tế; **[Đề xuất]** thiết kế của A13; **[Giả định]** chưa kiểm chứng; **[Cần người dùng phê duyệt]** quyết định cần owner.

---

## 1. Mục đích

Định nghĩa "tổ chức" của AI WORKSPACE: 18 vai trò agent mặc định (seed data), ranh giới trách nhiệm giữa các nhóm, Agent Registry động (thêm/xóa/thay thế vai trò ở runtime), ánh xạ role → skill, phân biệt agent template và agent instance, và tách bạch ba loại identity (Agent / Provider / User). Tài liệu là đầu vào cho module `agents`, `organization` (NestJS) và cho seed migration của A08 (DATABASE_DESIGN).

## 2. Phạm vi

- Trong phạm vi: catalogue 18 role, department mapping, permission level, file-ownership pattern, bảng ranh giới trách nhiệm, thiết kế registry (schema logic, API contract ở mức mô tả), identity model.
- Ngoài phạm vi: workflow engine/DAG (A07), schema SQL chi tiết (A08), sandbox/security enforcement (A09), UI tổ chức (A11). Scheduler và resource dùng role metadata ở đây nhưng mô tả riêng ở `AGENT_SCHEDULER.md`.

## 3. Nguyên tắc thiết kế [Đề xuất]

1. **Role là dữ liệu, không phải code.** 18 role chỉ là seed; hệ thống chạy được với 1 role hoặc 40 role. Không có `switch(role)` trong business logic; mọi hành vi lấy từ `AgentTemplate` (skills, provider policy, permissions, ownership patterns).
2. **Một task — một agent chịu trách nhiệm.** Agent khác chỉ hỗ trợ (review/help) qua message bus, không đồng sở hữu task.
3. **Ownership theo file pattern.** Mỗi role có danh sách glob được phép ghi; scheduler dùng pattern này để tránh hai agent ghi cùng file (quy tắc trong WORK_PLAN.md mục 3).
4. **Provider là thuộc tính có thể thay thế.** Agent "Backend Dev" hôm nay chạy Claude Code, mai chạy Codex: template chỉ ghi *provider suggestion*, instance ghi *provider binding*.
5. **Giới hạn đồng thời tính trên agent đang executing**, không tính agent đã tạo. Có thể tạo 18 instance nhưng chỉ `max_concurrent_agents` (default 5) được chạy cùng lúc (xem `RESOURCE_MANAGEMENT.md`).

## 4. Catalogue 18 vai trò mặc định [Đề xuất]

Ký hiệu permission level (xem mục 6): **L0** read-only, **L1** write-docs, **L2** write-code (sandbox workspace), **L3** write-code + run commands/tests, **L4** infra/deploy (luôn cần user approval từng lệnh nguy hiểm).

Provider suggestion là gợi ý mặc định, không ràng buộc. Hiện chỉ Claude Code CLI được cài [Đã xác minh]; Codex CLI chưa cài nên mọi gợi ý "Codex" là [Giả định] cho đến khi A06 xác nhận.

| # | Role (code) | Department | Trách nhiệm chính | Outputs tiêu biểu | Default skills | Provider gợi ý | Perm |
|---|---|---|---|---|---|---|---|
| 01 | `pm` Project Manager | Management | Lập WBS, ưu tiên, theo dõi tiến độ, điều phối họp, báo cáo owner. Không tự chạy workflow — chỉ đề xuất cho orchestrator (xem `AGENT_COLLABORATION.md` §9) | WORK_PLAN, PROJECT_STATUS, risk log, sprint report | planning, wbs, risk-management, reporting | Claude (Sonnet-class) | L1 |
| 02 | `ba` Business Analyst | Business & Architecture | Thu thập yêu cầu, PRD/SRS, user story, acceptance criteria | PRD.md, SRS.md, user stories, glossary | requirements, user-story, acceptance-criteria | Claude | L1 |
| 03 | `architect` Software Architect | Business & Architecture | Kiến trúc hệ thống, ADR, API contract, chuẩn code | SYSTEM_ARCHITECTURE, ADR/*, OpenAPI, event schema | architecture, adr, api-design, c4 | Claude (Opus-class cho ADR) | L1 |
| 04 | `uiux` UI/UX Designer | Design & UX | Design system, wireframe, token, component spec | DESIGN_SYSTEM, wireframes, token JSON | design-system, wireframing, accessibility | Claude | L1 |
| 05 | `frontend` Frontend Developer | Development | Triển khai UI (React/TS), state, integration với API | `apps/web/**`, component tests | react, typescript, tailwind, vitest | Claude hoặc Codex | L3 |
| 06 | `backend` Backend Developer | Development | NestJS modules, services, API, business logic | `apps/api/src/**`, unit tests | nestjs, typescript, drizzle, vitest | Claude hoặc Codex | L3 |
| 07 | `db_infra` DB & Infra Engineer | Infrastructure | Schema, migration, seed, docker-compose, Redis/Postgres | `apps/api/src/database/**`, `drizzle/**`, `docker/**`, migrations | sql, drizzle, docker-compose, redis | Claude | L3 |
| 08 | `qa` QA Engineer | Quality & Security | Test plan, test case, manual verification, bug report | TEST_PLAN, test cases, bug tickets | test-design, exploratory-testing, bug-reporting | Claude | L1 (L3 khi chạy test) |
| 09 | `security` Security Engineer | Quality & Security | Threat model, secure coding review, secret scanning, sandbox policy | SECURITY.md, threat model, findings | threat-modeling, owasp, secret-scan | Claude | L1 (read code) |
| 10 | `reviewer` Code Reviewer & Product Evaluator | Quality & Security | Review diff, đánh giá sản phẩm so với PRD, gate "COMPLETED" | review report, evaluation scorecard | code-review, product-evaluation | Claude (Opus-class) | L0/L1 |
| 11 | `research` Technical Research / R&D | Research | Khảo sát công nghệ, so sánh, PoC nhỏ | `docs/research/*`, PoC branch | web-research, benchmark, poc | Claude | L1 (L2 cho PoC) |
| 12 | `integration` System Integration Engineer | Integration | Provider adapters, MCP, webhook, third-party API | adapter code, integration tests | api-integration, mcp, sdk | Claude hoặc Codex | L3 |
| 13 | `performance` Performance & Scalability | Performance | Benchmark, profiling, concurrency, capacity planning | perf reports, load test scripts | profiling, load-testing, capacity-planning | Claude | L3 |
| 14 | `devsecops` DevSecOps & Infra | Infrastructure | CI/CD, container hardening, secret mgmt, deploy script | `.github/**`, Dockerfile, pipeline | github-actions, docker, sast, iac | Claude | L4 |
| 15 | `docs` Documentation & Knowledge | Knowledge | Tài liệu người dùng/dev, knowledge base, changelog | docs/**, CHANGELOG, README | technical-writing, diataxis, changelog | Claude (Haiku/Sonnet-class) | L1 |
| 16 | `ux_research` UX Research & Product Designer | Design & UX | Persona, journey, usability test, product discovery | personas, journey maps, usability findings | user-research, journey-mapping, survey | Claude | L1 |
| 17 | `test_automation` Test Automation & Browser | Quality & Security | E2E (Playwright), browser automation, visual regression | `e2e/**`, CI test jobs, screenshots | playwright, e2e, visual-regression | Claude hoặc Codex | L3 |
| 18 | `release` Release Manager & Product Acceptance | Management | Release checklist, versioning, acceptance ký tên, rollback plan | RELEASE_NOTES, acceptance report, tag | release-mgmt, semver, acceptance | Claude | L1 (L4 khi tag/deploy, cần approval) |

**Ghi chú về model tier** [Giả định]: "Opus/Sonnet/Haiku-class" là gợi ý theo độ khó; cấu hình thật nằm trong `ProviderProfile.model`. Tên model cụ thể chốt sau khi A05 xác nhận model nào khả dụng trên tài khoản. Thực nghiệm của A13 cho thấy CLI mặc định hiện dùng `claude-fable-5-1` cho main model và `claude-haiku-5-5` cho tác vụ phụ [Đã xác minh từ trường `modelUsage` của output JSON].

### 4.1 File ownership patterns mặc định [Đề xuất]

| Role | Được ghi (glob) | Chỉ đọc |
|---|---|---|
| pm | `WORK_PLAN.md`, `PROJECT_STATUS.md`, `docs/product/plan/**` | toàn bộ |
| ba | `docs/product/**` | toàn bộ |
| architect | `docs/architecture/**`, `docs/adr/**`, `contracts/**` | toàn bộ |
| uiux, ux_research | `docs/design/**`, `packages/design-tokens/**` | `apps/web/**` |
| frontend | `apps/web/**`, `packages/ui/**` | `contracts/**` |
| backend | `apps/api/**`, `packages/shared/**` | `contracts/**`, `apps/api/src/database/**`, `drizzle/**` |
| db_infra | `apps/api/src/database/**`, `drizzle/**`, `docker/**`, `infra/local/**` | `apps/api/**` (trừ `apps/api/src/database/**`) |
| qa | `docs/qa/**`, `tests/manual/**` | toàn bộ |
| security | `docs/security/**` | toàn bộ |
| reviewer | `docs/reviews/**` | toàn bộ |
| research | `docs/research/**`, `poc/**` | toàn bộ |
| integration | `apps/api/src/providers/**`, `packages/adapters/**` | `contracts/**` |
| performance | `docs/performance/**`, `bench/**` | toàn bộ |
| devsecops | `.github/**`, `Dockerfile*`, `infra/**` | toàn bộ |
| docs | `docs/**` (trừ thư mục role khác đang sở hữu trong task đang chạy), `README.md`, `CHANGELOG.md` | toàn bộ |
| test_automation | `e2e/**`, `tests/e2e/**` | `apps/**` |
| release | `RELEASE_NOTES.md`, `docs/release/**` | toàn bộ |

Quy tắc xung đột: khi hai pattern chồng lấn (ví dụ `docs` và `research` đều có `docs/research/**`), scheduler **ưu tiên owner của task đang RUNNING**; role khác bị chặn ghi cho đến khi task đó kết thúc (chi tiết: `AGENT_SCHEDULER.md` §5 "workspace availability").

## 5. Bảng ranh giới trách nhiệm theo nhóm [Đề xuất]

| Nhóm | Role | Quyết định được tự chốt | Phải xin ý kiến | Không được làm |
|---|---|---|---|---|
| Management | 01 pm, 18 release | Thứ tự ưu tiên trong sprint, nội dung báo cáo | Thay đổi scope, deadline, bật/tắt provider | Trực tiếp sửa code; gán task vượt quyền orchestrator |
| Business & Architecture | 02 ba, 03 architect | Nội dung PRD/SRS, ADR, API contract | Thay đổi contract đã publish (phải thông báo Development) | Sửa code implementation |
| Design & UX | 04 uiux, 16 ux_research | Token, component spec, flow | Thay đổi ảnh hưởng contract API | Sửa `apps/web` trực tiếp (chỉ đề xuất cho 05) |
| Development | 05 frontend, 06 backend | Chi tiết implementation trong vùng sở hữu | Thay đổi schema DB (07), contract (03) | Ghi ngoài ownership pattern; tự merge |
| Quality & Security | 08 qa, 09 security, 10 reviewer, 17 test_automation | Verdict PASS/FAIL, severity | Chặn release (phải qua 18 + owner) | Sửa code sản phẩm (chỉ tạo ticket / patch đề xuất) |
| Research | 11 research | Kết luận nghiên cứu, PoC | Đưa PoC vào main | Chỉnh code sản phẩm |
| Integration | 12 integration | Adapter internals | Thay đổi event schema | Bỏ qua sandbox policy của 09 |
| Performance | 13 performance | Benchmark method, ngưỡng cảnh báo | Thay đổi concurrency limit global (owner) | Tự tăng `max_concurrent_agents` |
| Infrastructure | 07 db_infra, 14 devsecops | Local infra, pipeline | Mọi thao tác deploy/destroy (approval từng lệnh) | Chạy lệnh L4 không có approval |
| Knowledge | 15 docs | Cấu trúc tài liệu | Ghi đè doc của role khác đang RUNNING | Thay đổi nội dung quyết định (ADR) |

## 6. Permission level [Đề xuất] — [Cần người dùng phê duyệt]

| Level | Quyền | Cơ chế thực thi (phối hợp A09) |
|---|---|---|
| L0 | Đọc workspace | CLI chạy với `--permission-mode plan` hoặc tool allowlist chỉ Read/Grep/Glob |
| L1 | L0 + ghi file khớp ownership pattern (docs) | allowlist Write/Edit + hook kiểm pattern (PreToolUse hook) |
| L2 | L1 + ghi code trong workspace sandbox (worktree riêng) | worktree per task; không có quyền chạy lệnh ngoài `git`, package manager |
| L3 | L2 + chạy test/build/dev server | Bash allowlist theo lệnh; timeout per command |
| L4 | L3 + lệnh infra (docker, deploy, git push/tag) | mỗi lệnh L4 tạo `ApprovalRequest`; agent bị PAUSED tới khi user duyệt |

Permission level của instance không được vượt template; user có thể hạ thấp per-instance hoặc per-task.

## 7. Agent Registry — thiết kế động [Đề xuất]

### 7.1 Khái niệm

- **AgentTemplate** (còn gọi role): bản mô tả vai trò — tên, department, mô tả, system prompt skeleton, `skills[]`, `providerPolicy`, `permissionLevel`, `ownershipPatterns[]`, `defaultModelTier`, `estimatedCostTier`, `version`, `isSeed`, `isArchived`.
- **AgentInstance** (còn gọi agent): một "nhân viên" cụ thể được tạo từ template trong một organization — có `displayName`, `avatar`, `templateId`, `templateVersion` (pin), `providerBinding` (provider + model + credential ref), `overrides` (prompt bổ sung, skill thêm/bớt, permission thấp hơn), `status` (ACTIVE/DISABLED/ARCHIVED), `stats`.
- **AgentExecution**: một lần instance chạy một task (CLI subprocess) — sở hữu bởi scheduler, không phải registry.

Quan hệ: Template 1—N Instance 1—N Execution. Xóa template → instance chuyển `templateId` sang bản archived (không mồ côi). Thay thế role = tạo template mới version+1, instance chọn thời điểm "upgrade".

### 7.2 Seed và tùy biến

- 18 template trong mục 4 được seed với `isSeed=true`; user có thể **archive** (không xóa cứng) hoặc **clone** để sửa.
- Thêm role mới: `POST /org/{orgId}/agent-templates` với body đầy đủ; validation: `skills[]` phải tồn tại trong Skill Catalogue, `ownershipPatterns` phải là glob hợp lệ và không chứa `..`.
- Replace: `PUT /agent-templates/{id}` tạo version mới; instance có `autoFollowTemplate=true` sẽ nhận version mới khi **không có execution đang chạy**.

### 7.3 Role → Skill mapping

- **Skill Catalogue** là bảng riêng: `Skill { code, name, description, kind: 'prompt' | 'claude-skill-dir' | 'mcp-server' | 'tool-allowlist', payloadRef, providerCompat[] }`.
- Với Claude Code CLI, skill kiểu `claude-skill-dir` được materialize vào `.claude/skills/<code>/SKILL.md` của workspace task trước khi spawn (chi tiết cơ chế spawn: `CLAUDE_CODE_INTEGRATION.md` của A05). Với Codex, skill kiểu `prompt` được ghép vào instructions [Giả định — chờ A06].
- `RoleSkill { templateId, skillCode, required: boolean, weight: 0..1 }`. `weight` dùng trong scoring của scheduler (`AGENT_SCHEDULER.md` §5).
- Task cũng khai `requiredSkills[]`/`preferredSkills[]`; so khớp theo code (không fuzzy).

### 7.4 Ví dụ dữ liệu (minh họa, không phải schema cuối — schema do A08)

```ts
// AgentTemplate (seed)
{
  code: 'backend', name: 'Backend Developer', department: 'Development',
  version: 1, isSeed: true,
  skills: [{ code: 'nestjs', required: true, weight: 1 }, { code: 'drizzle', required: false, weight: 0.6 }],
  providerPolicy: { preferred: ['claude-code'], allowed: ['claude-code', 'codex'], switchRequiresApproval: true },
  permissionLevel: 'L3',
  ownershipPatterns: ['apps/api/**', 'packages/shared/**'],
  defaultModelTier: 'standard',
  promptSkeleton: 'You are the Backend Developer of {{org.name}} ...'
}
// AgentInstance
{
  id: 'agt_01J...', orgId: 'org_1', templateId: 'tpl_backend', templateVersion: 1,
  displayName: 'Minh (Backend)', status: 'ACTIVE',
  providerBinding: { provider: 'claude-code', model: null /* provider default */, credentialRef: 'user-oauth' },
  overrides: { permissionLevel: 'L2', extraSkills: [], promptAppend: 'Prefer Vietnamese comments.' },
  limits: { maxConcurrentExecutions: 1, dailyBudgetUsd: 5 }
}
```

## 8. Agent Identity vs Provider Identity vs User Identity [Đề xuất]

| Identity | Là gì | Nguồn sự thật | Dùng cho |
|---|---|---|---|
| **User Identity** | Người dùng app (owner, operator) — đăng nhập AI WORKSPACE | bảng `User`, auth của app (A09) | Phê duyệt, audit "ai ra lệnh", RBAC UI |
| **Agent Identity** | `AgentInstance.id` — danh tính logic của "nhân viên AI" | bảng `AgentInstance` | Gán task, message bus, ownership, thống kê hiệu năng, log/trace |
| **Provider Identity** | Tài khoản/credential tại provider (Claude.ai OAuth của user, API key, ChatGPT login của Codex) | `ProviderCredential` (encrypted, A09) + `ProviderProfile` | Spawn subprocess, quota/rate-limit accounting, cost |

Nguyên tắc:
1. Một Provider Identity có thể phục vụ nhiều Agent Identity (hiện tại: **một** tài khoản claude.ai `team` phục vụ tất cả agent — [Đã xác minh] qua `claude auth status`: `authMethod: claude.ai`, `subscriptionType: team`). Hệ quả: quota 5-giờ (nếu áp dụng — [Giả định], xem `RESOURCE_MANAGEMENT.md` §8) là **chia sẻ** giữa 18 agent; scheduler phải tính per-provider-credential, không per-agent.
2. Agent Identity không bao giờ chứa secret. `credentialRef` chỉ là con trỏ.
3. Audit log luôn ghi bộ ba `(userId: ai ra lệnh/phê duyệt, agentId: ai thực thi, providerCredentialId: bằng tài khoản nào)`.
4. Session id của CLI (`session_id` trong output JSON, ví dụ `805a9d1e-f2af-4c7d-949e-5aa86dcc0eac` [Đã xác minh]) là **Execution evidence**, lưu trong `AgentExecution.providerSessionId` để resume/trace, không phải identity.
5. User có thể gắn credential riêng cho từng instance (ví dụ agent Codex dùng tài khoản ChatGPT khác) — mỗi credential là một "pool" riêng trong resource manager.

## 9. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Role = seed data trong DB, không enum trong code | Owner yêu cầu thêm/xóa/thay thế; tránh deploy khi đổi tổ chức | [Đề xuất] |
| Template versioning + instance pin | Cho phép đổi role mà không phá execution đang chạy | [Đề xuất] |
| Ownership theo glob, kiểm bằng hook trước khi ghi | Khớp quy tắc "không hai agent ghi cùng file" đã có trong WORK_PLAN | [Đề xuất] |
| 5 permission level thay vì bool "có thể chạy lệnh" | Hợp với permission-mode của Claude Code và approval flow L4 | [Đề xuất] — [Cần người dùng phê duyệt] |
| Tách 3 identity | Một credential dùng chung cho nhiều agent (xác minh thực tế) nên quota/cost phải tính theo credential | [Đã xác minh] động cơ, [Đề xuất] thiết kế |
| Reviewer (10) và Release (18) là role riêng, không gộp vào QA | Gate "COMPLETED cần bằng chứng" (WORK_PLAN §3) cần chủ thể độc lập với người viết test | [Đề xuất] |

## 10. Dependencies

- A05 `CLAUDE_CODE_INTEGRATION.md`: cách truyền skill/permission vào subprocess (`--permission-mode`, `--allowedTools`, `.claude/skills`).
- A06 `CODEX_INTEGRATION.md`: tương đương cho Codex (hiện chưa cài CLI).
- A07 `ORCHESTRATION_WORKFLOW_RESEARCH.md`: workflow engine tiêu thụ `AgentInstance` qua scheduler.
- A08 `DATABASE_DESIGN.md`: bảng `AgentTemplate`, `AgentInstance`, `Skill`, `RoleSkill`, `ProviderCredential`.
- A09 `SECURITY.md`: enforce permission level, encrypt credential.
- A11 `DESIGN_SYSTEM.md`: UI quản lý org/agent.

## 11. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| 18 role quá nhiều cho dự án nhỏ → chi phí prompt/overhead | Tốn quota, chậm | Default org chỉ **activate** 5–7 instance; còn lại ở trạng thái DISABLED (template vẫn seed) |
| Ownership glob chồng lấn gây deadlock ghi | Agent chờ nhau | Validation khi tạo template cảnh báo overlap; scheduler có timeout chờ workspace |
| Role mơ hồ (ví dụ 10 reviewer vs 08 qa) | Task bị gán sai | Bảng ranh giới §5 + scoring `AGENT_SCHEDULER.md`; PM agent được quyền đề xuất gán lại |
| Template thay đổi khi execution đang chạy | Prompt không nhất quán | Pin `templateVersion` trong execution |
| Mỗi subprocess load toàn bộ skills/MCP của user (đo được ~48k token prompt cache [Đã xác minh]) | Cost per spawn cao | Spawn với `CLAUDE_CONFIG_DIR` riêng/`--settings` tối giản — chi tiết `RESOURCE_MANAGEMENT.md` §4.3 |

## 12. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Có cho phép user tạo template với permission L4 không, hay L4 chỉ seed-only?
2. [Cần người dùng phê duyệt] Instance có được bind nhiều provider cùng lúc (failover tự động) không? A13 đề xuất **không** — chuyển provider cần approval (xem `RESOURCE_MANAGEMENT.md` §9).
3. Skill Catalogue: ai sở hữu nội dung SKILL.md — bundled trong repo hay user upload? (liên quan A09 vì skill là code chạy trong CLI).
4. Model tier cụ thể cho từng role phụ thuộc model khả dụng trên tài khoản — chờ A05.
5. Multi-org: một user có nhiều organization → registry scope theo `orgId` (đề xuất có, nhưng Phase 1 chỉ 1 org).

## 13. Acceptance Criteria (cho Phase 1 khi triển khai)

- [ ] Seed migration tạo đúng 18 `AgentTemplate` với `isSeed=true`, mỗi template có ≥1 skill required và ≥1 ownership pattern.
- [ ] API: tạo/clone/archive template; tạo/disable instance; validation glob và skill code; test tự động.
- [ ] Không có `switch` trên role code trong `apps/api/src` (lint rule hoặc review checklist).
- [ ] Pin `templateVersion` trong execution; thay đổi template không ảnh hưởng execution đang RUNNING (integration test).
- [ ] Audit log ghi đủ bộ ba identity cho mọi execution.
- [ ] Hook ownership chặn ghi ngoài pattern, có test với 2 instance chồng lấn pattern.
- [ ] Tài liệu này được owner duyệt mục 6 (permission) và mục 12 (câu 1, 2).
