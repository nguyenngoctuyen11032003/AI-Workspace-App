# BACKEND_ARCHITECTURE — Kiến trúc backend NestJS

> Tác giả: System Architect · Phase 0 (chỉ tài liệu) · Draft v0.1 · 2026-10-30
> Nhãn: **[Đã xác minh]** (có nguồn) · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**. Các D-xxx đều "Chờ duyệt" (`DECISION_LOG.md`).
> Liên quan: `docs/SYSTEM_ARCHITECTURE.md`, `docs/API_DOCUMENTATION.md`, `docs/DATABASE_DESIGN.md`, `docs/SECURITY.md`.

---

## 1. Mục đích

Định nghĩa cấu trúc `apps/api` (NestJS): danh sách module và trách nhiệm, phân lớp, DTO/validation, error format, config, logging, queue, WebSocket gateway và các seam kiểm thử, đủ để nhiều agent backend làm song song mà không giẫm chân nhau.

## 2. Phạm vi

- **Trong:** module map, layering, cross-module rules, DTO/validation, errors, config, logging, queues, gateways, testing seams.
- **Ngoài:** schema bảng (xem `DATABASE_DESIGN.md`), danh mục endpoint (xem `API_DOCUMENTATION.md`), threat model (xem `SECURITY.md`), mã nguồn.

## 3. Nội dung chi tiết

### 3.1 Cấu trúc thư mục `apps/api/src`

```
src/
  main.ts                 # API entry (HTTP + WS)
  worker.ts               # Worker entry (BullMQ processors), dùng chung AppModule con
  app.module.ts
  common/                 # filter, interceptor, guard, pipe, decorator, tenant-context, logger, config
  database/               # schema/ (Drizzle), migrations/, seeds/, drizzle.module.ts   (DATABASE_DESIGN §1.3)
  modules/
    auth/  organizations/  departments/  agents/  skills/  providers/
    sessions/  orchestrator/  scheduler/  workspace/  sandbox/  approvals/
    events/  usage/  office/  audit/  projects/  tasks/  notifications/  settings/
```

**[Đề xuất]** `projects`, `tasks`, `notifications`, `settings` không nằm trong danh sách module Owner/Orchestrator yêu cầu nhưng được thêm vì NAVIGATION_STRUCTURE (Task Center, Activity, Settings, Projects) và DATABASE_DESIGN (bảng `projects`, `tasks`, `notifications`, `system_settings`) cần chủ sở hữu rõ ràng. `orchestrator` gồm `definition/ run/ revision/ bus/ assignment/` theo `ORCHESTRATION_WORKFLOW_RESEARCH.md` §10; `scheduler` tách riêng vì `AGENT_SCHEDULER.md` §3 mô tả nó là module có tick loop riêng.

### 3.2 Module list và trách nhiệm

| Module | Trách nhiệm | Bảng sở hữu (DATABASE_DESIGN) | Phụ thuộc |
|---|---|---|---|
| `auth` | Chế độ desktop: bearer token per-launch + inject user `local`; chế độ server: JWT/session; `TenantContext`; guard/RBAC | `users`, `organization_members` | organizations |
| `organizations` | CRUD org/workspace, thành viên, seed org `local` | `organizations`, `organization_members` | — |
| `departments` | 12 phòng ban mặc định (seed), CRUD, head, ngân sách, org chart | `departments` | organizations |
| `agents` | CRUD agent (Agent Builder), lifecycle 12 trạng thái, profile, gán skill, presence; 18 vai trò là seed (D-016) | `agents`, `agent_profiles`, `agent_provider_connections`, `agent_skills`, `agent_memories` | departments, skills, providers |
| `skills` | Skill Library, version, document, Teach Agent flow, nạp skill (`--add-dir`, junction) (D-013) | `skills`, `skill_versions`, `skill_documents` | agents |
| `providers` | Provider Hub; `AgentAdapter` cho Claude Code / Codex / OpenHands / generic CLI; health check; ListModels; Connect/Disconnect | `provider_configs` | agent-core, keychain |
| `sessions` | Vòng đời `agent_session`, nhận `AgentEvent` từ adapter, ghi `agent_events`, replay | `agent_sessions`, `agent_events`, `agent_messages` | providers, events |
| `orchestrator` | Conductor: validate Workflow Definition, RunService, XState guard, revision loop, message bus, PM assignment | `workflows`, `workflow_versions`, `workflow_nodes`, `workflow_edges`, `workflow_runs`, `node_runs` | scheduler, approvals, events |
| `scheduler` | `tick(run)` ready-set, priority, dispatcher, semaphore Lua (global + provider + agent + worktree), retry/backoff, heartbeat, stale recovery | `task_assignments`, `resource_limits` (đọc) | orchestrator, Redis |
| `projects` / `tasks` | Project, revision, task, dependency, kanban data | `projects`, `project_members`, `project_revisions`, `tasks`, `task_dependencies` | workspace |
| `workspace` | Git bare repo, worktree per task, ownership manifest, diff validation, integration branch, cleanup (D-009) | `git_repositories`, `git_worktrees`, `artifacts` | `packages/workspace` |
| `sandbox` | Sandbox Executor: BullMQ `sandbox`, dockerode, limits, network none/allowlist, capture log/report (D-007) | (dùng `artifacts`) | workspace, Docker |
| `approvals` | Policy engine, approval lifecycle, hook endpoint `/hooks/*` (listener riêng), cached decision, `input_hash` | `approval_policies`, `approvals` | events, sessions |
| `events` | Outbox relay, Redis pub/sub + Stream, Socket.IO gateways, replay `lastEventId` (D-014) | `agent_events` (ghi qua sessions), outbox | Redis |
| `usage` | Ghi `usage_records` cùng tx với `llm.response`, rollup 5 phút, budget check Redis | `usage_records`, `usage_daily_rollups` | events |
| `office` | Office Projector: subscribe stream, giữ `office_state`, write-behind `character_positions`; layout/room/furniture CRUD | `office_layouts`, `office_rooms`, `office_furniture`, `desks`, `character_positions` | events |
| `audit` | `AuditInterceptor`, truy vấn `audit_logs`, redaction | `audit_logs` | common |
| `notifications` / `settings` | Taxonomy thông báo (NAVIGATION §10), preferences, `system_settings` | `notifications`, `system_settings` | events |

**[Đã xác minh]** tên bảng lấy từ danh sách 44 bảng ở `docs/DATABASE_DESIGN.md` §4. Ghi chú: danh sách này **không chứa bảng `outbox`** dù D-014 và ORCHESTRATION §7.3 yêu cầu outbox giao dịch, và §12 mục 4/7 coi outbox / `event_sequences` là điểm mở. Đề xuất thêm bảng `event_outbox` (xem §3.9) — **[Cần người dùng phê duyệt]** vì làm số bảng thành 45.

### 3.3 Phân lớp (layering) trong mỗi module **[Đề xuất]**

```
Controller / Gateway   -> parse + validate DTO, authz guard, trả response DTO. Không chứa logic.
Application Service    -> use case, transaction boundary, phát domain event. Gọi service của module khác qua interface export.
Domain (pure TS)       -> state machine, scoring, policy evaluation, manifest matching. Không import Nest/Drizzle.
Repository             -> Drizzle query, luôn thêm `organization_id` từ TenantContext. Cô lập relational API (DATABASE_DESIGN §1.2).
Infrastructure adapter -> Redis, BullMQ, Docker, Git, keychain, CLI process. Ẩn sau interface/token DI.
```

Quy tắc:
1. Controller → Service → Repository, không bỏ lớp, không gọi ngược lên.
2. Module không import repository của module khác; dùng service export hoặc domain event.
3. Domain pure để test không cần DB (ví dụ `assignmentScore()`, `canTransition()`).
4. Mọi hành động đổi trạng thái chạy trong **một transaction**: cập nhật aggregate + `appendEvent` + `outbox` (DATABASE_DESIGN §5, §6).

### 3.4 DTO và validation

- **[Đề xuất]** Zod là nguồn duy nhất, đặt trong `packages/contracts`. DTO request/response/event suy ra bằng `z.infer`. Trong NestJS dùng `ZodValidationPipe` global (hoặc `nestjs-zod`) thay `class-validator`, vì DATABASE_DESIGN §0 và §5.1 đã dùng Zod cho JSONB và payload event; một thư viện, hai nơi dùng (FE form + BE).
- OpenAPI sinh từ Zod (xem `API_DOCUMENTATION.md` §9).
- Quy ước: `camelCase` JSON, ID là chuỗi uuid v7 (ULID chỉ cho `taskId` thư mục worktree theo WORKSPACE_AND_GIT_ISOLATION §4), thời gian ISO-8601 UTC, tiền tệ là chuỗi decimal (tránh float), không trả cột nội bộ (`deleted_at`, `credential_ref`).
- `whitelist/forbidNonWhitelisted`: Zod `.strict()` cho request body.
- Payload event kiểm tra Zod khi ghi (`schema_version` trong payload).

### 3.5 Error format

Một envelope duy nhất do `AllExceptionsFilter` tạo (chi tiết ở `API_DOCUMENTATION.md` §3.2), dạng RFC 9457 rút gọn:

```json
{ "error": { "code": "APPROVAL_EXPIRED", "message": "...", "details": [...], "correlationId": "01J...", "status": 409 } }
```

- Mã lỗi là enum trong `packages/contracts/errors.ts`; `message` không chứa stack trace, path nội bộ hay secret (`SECURITY.md` §7: không expose stack).
- Domain error (class `DomainError`) map sang HTTP status ở một chỗ. Lỗi provider (`PROVIDER_UNAVAILABLE`, `RATE_LIMITED`, `AUTH_EXPIRED`) giữ `retryAfterMs`.
- Redaction middleware chạy trước khi ghi log/audit (`SECURITY.md` §4.4).

### 3.6 Config

- **[Đề xuất]** `@nestjs/config` + schema Zod validate lúc boot; thiếu/sai thì process thoát với thông báo rõ. Tách `ConfigModule` có nhóm: `app` (`APP_MODE=desktop|server`, port), `db`, `redis`, `security` (token, origin allowlist, rate limit), `providers`, `workspace` (`WORKSPACE_ROOT`), `sandbox` (limits mặc định), `features`.
- **[Đã xác minh]** `APP_MODE` và `.env.example` đã có trong `docs/DATABASE_DESIGN.md` §8, §11.
- Giới hạn runtime (concurrency, budget, retention) không nằm trong env mà trong `system_settings`/`resource_limits` để đổi lúc chạy; mặc định concurrency 5, trần 18 (D-010).
- Secret chỉ là `credential_ref`; giá trị lấy từ OS keychain (`@napi-rs/keyring`/`keytar`) hoặc Vault/env ở server (DATABASE_DESIGN §8; D-008).

### 3.7 Logging (pino)

- **[Đề xuất]** `nestjs-pino`, JSON ra stdout, mức theo env; mỗi request có `correlationId` (header `X-Correlation-Id` hoặc sinh mới), đưa vào `AsyncLocalStorage` cùng `organizationId`, `userId`; worker gắn `runId/nodeRunId/sessionId/agentId`.
- Redact path: `req.headers.authorization`, `*.token`, `*.credential*`, `*.systemPrompt` (chỉ hash + length, theo DATABASE_DESIGN §7).
- Log nghiệp vụ không thay thế `agent_events`/`audit_logs`; log là chẩn đoán, event là nguồn sự thật.
- Output thô của agent không vào log chung: ghi artifact `type = log` (DATABASE_DESIGN §5.1).

### 3.8 Queue (BullMQ)

**[Đã xác minh]** Danh sách queue từ `DATABASE_DESIGN.md` §10 và ORCHESTRATION §2: `agent-runner`, `workflow-engine`, `usage-rollup` (5m), `partition-maintainer` (daily 02:00), `event-retention` (daily 03:00), `approval-expiry` (delayed), `presence-reaper` (30s), `office-tick` (100 ms, chỉ khi có client), cộng `scheduler` (1 job/run, `jobId = runId`), `node.tool`, `node.test`/`node.deploy`, `sandbox`.

| Queue | Processor | Ghi chú |
|---|---|---|
| `scheduler` | `SchedulerProcessor.tick(runId)` | idempotent, `SELECT FOR UPDATE` + jobId dedupe |
| `node.agent` / `agent-runner` | `AgentRunnerProcessor` | concurrency đọc semaphore; spawn qua adapter; heartbeat vào Redis |
| `node.tool` | tool node | concurrency 20 |
| `sandbox` | `SandboxProcessor` (file riêng, sandboxed processor) | CPU-bound không kéo API |
| cron: `usage-rollup`, `partition-maintainer`, `event-retention`, `presence-reaper`, `approval-expiry` | maintenance | |

Quy tắc **[Đề xuất]**: tên queue tập trung ở một enum trong `common/queues.ts`; job payload chỉ chứa ID (không chứa dữ liệu lớn); mọi job có `attempts`, `backoff` exponential + jitter; job lỗi cuối cùng ghi event `*.failed`; trên Windows kill cây tiến trình bằng `tree-kill`/`taskkill /T /F` (ORCHESTRATION §10).
**Mâu thuẫn:** ORCHESTRATION §2 đặt `node.agent` concurrency 10 và semaphore global 10; D-010 đặt 5 mặc định. Theo D-010: concurrency worker = `min(resource_limits.max_concurrent_agents, 18)`, giá trị khởi đầu 5.

### 3.9 Events, outbox và realtime gateway

- **Ghi:** `EventService.append(tx, event)` cấp `sequence` per aggregate trong cùng transaction, ghi `agent_events` + `event_outbox`. Role DB của app chỉ INSERT/SELECT `agent_events` (DATABASE_DESIGN §5.1).
- **Phát:** `OutboxRelay` poll 250 ms `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 500` → `PUBLISH aiw:ch:*` và `XADD` stream, đánh dấu đã publish (ORCHESTRATION §7.3).
- **Ephemeral:** `llm.stream_chunk`, `AGENT_OUTPUT_CHUNK`, `office.animation` không qua outbox, publish thẳng Redis (DATABASE_DESIGN §5.2, ORCHESTRATION §7.3).
- **Replay:** `GET /sessions/:id/events?afterSeq=`, Redis Stream buffer ~2000 sự kiện, reconnect < 10 phút (DATABASE_DESIGN §5.4); toàn cục bằng `lastEventId`/`global_seq`.
- **[Cần người dùng phê duyệt]** Outbox đầy đủ cho cả desktop và server (một code path), thay vì publish-after-commit ở desktop như DATABASE_DESIGN §5.4/§12 mục 4 gợi ý. Lý do: giảm hai nhánh logic; chi phí là một bảng và độ trễ ≤ 250 ms.

**Gateway Socket.IO** (`@nestjs/platform-socket.io`, D-014): namespace `/realtime`, rooms `org:{id}`, `project:{id}`, `run:{id}`, `agent:{id}`, `session:{id}`, `approvals:{orgId}`, `office:{layoutId}`. Handshake auth bằng bearer (subprotocol `aiw-token.<token>` hoặc first-message trong 2 s, không để token ở query string) và Origin check (`SECURITY.md` §7). Message ≤ 256 KB, 3 lần auth sai thì ngắt. Lệnh điều khiển agent (pause/resume/cancel/inject_message) dùng ack, đi qua `aiw:ch:control:agent:{agentId}`. Chi tiết event envelope ở `API_DOCUMENTATION.md` §6.

### 3.10 Provider adapter

**[Đã xác minh]** Claude Code dùng `@anthropic-ai/claude-agent-sdk` `query()` (SDK kế thừa OAuth login, `apiKeySource: "none"`), cổng approval qua `canUseTool` + PreToolUse hook (`docs/research/CLAUDE_CODE_INTEGRATION.md` §0, §4.3, §7); Codex ưu tiên `codex app-server` stdio, fallback `codex exec --json` (`CODEX_INTEGRATION.md` §11.1); OpenHands qua `openhands-agent-server` (D-006).

**Mâu thuẫn tên interface:** `CLAUDE_CODE_INTEGRATION.md` §9.1 định nghĩa `BrainAdapter`/`BrainEvent` (`'claude-code' | 'codex-cli'`), còn `CODEX_INTEGRATION.md` §11.2 định nghĩa `AgentEvent` với `AgentBrain = 'codex' | 'claude'` trong `packages/agent-core`. **[Đề xuất]** hợp nhất thành `AgentAdapter` / `AgentEvent` trong `@aiw/agent-core`, `provider.kind ∈ {claude_code, codex, openhands, generic_cli}`; tên enum cuối do `provider_configs` quyết định.

Giao diện Provider Hub (PRD §5.4): `connect, disconnect, authenticate, checkConnection, listModels, createSession, resumeSession, sendMessage, streamEvents, cancelTask, getUsage, getError, getSessionHistory`. Hợp đồng test chung (contract test) chạy với fake adapter.

### 3.11 Workspace, sandbox, approvals (điểm kiến trúc)

- `workspace`: `repo.git` bare, `worktrees/<8 ký tự ULID>`, manifest do orchestrator sinh nằm ngoài worktree; sau khi agent xong, diff validation là cổng quyết định (D-009, `WORKSPACE_AND_GIT_ISOLATION.md` §4–7).
- `sandbox`: argv không qua shell, whitelist env, `network: none | egress-allowlist`, giới hạn cpus/memory/pids/timeout (`INFRASTRUCTURE_DESIGN.md` §6). Tránh Docker-in-Docker.
- `approvals`: hook endpoint là listener riêng với `AIW_HOOK_TOKEN`, giữ HTTP mở chờ quyết định (timeout 540 s), hết hạn là `deny`, `input_hash` SHA-256 chống TOCTOU (`SECURITY.md` §5.3, §7).

### 3.12 Testing seams

| Seam | Cách thay thế | Dùng cho |
|---|---|---|
| `AgentAdapter` | `FakeAgentAdapter` phát chuỗi `AgentEvent` kịch bản | test Conductor, runner, UI event mà không tốn quota |
| `Clock` | `ClockPort` (now, setTimeout) | test timeout, expiry, retry (mọi timeout dùng `now()` Postgres, ORCHESTRATION §10) |
| `GitPort` | tạo repo tạm trên tmp thật hoặc fake | test worktree, manifest, diff |
| `SandboxPort` | fake executor trả `SandboxResult` | test pipeline mà không cần Docker |
| `KeychainPort` | in-memory | test auth/credential_ref |
| `EventBusPort` | in-memory | test fan-out |
| Postgres/Redis | Testcontainers hoặc compose dev | integration test; migration chạy thật |
| Domain pure | unit test thuần | scoring, state machine, policy |

Phân tầng test: unit (domain) > integration (module + DB + Redis) > contract (adapter, `packages/contracts` so khớp OpenAPI snapshot) > e2e (Playwright, `e2e/`) > chaos (kill -9 worker, restart Redis/Postgres, 2 scheduler đồng thời: ORCHESTRATION §10). Mỗi module có test về cô lập tenant (không rò dữ liệu `organization_id` khác).

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| B-1 | Zod làm DTO/validation, không class-validator | Một schema cho BE, FE, event, OpenAPI | [Đề xuất] |
| B-2 | Repository mỏng bao Drizzle | Cô lập relational API kém trưởng thành | [Đã xác minh] DATABASE_DESIGN §1.2 |
| B-3 | Mọi mutation = aggregate + event + outbox trong một tx | Không mất event, replay đúng | [Đề xuất] dựa D-014 |
| B-4 | Postgres là source of truth, Redis tạm | Recovery sau crash | [Đã xác minh] AGENT_SCHEDULER §3 |
| B-5 | Hợp nhất `AgentAdapter` (thay `BrainAdapter`) | Tránh hai interface cho cùng việc | [Đề xuất] |
| B-6 | Thêm bảng `event_outbox` | D-014 cần, danh sách 44 bảng chưa có | [Cần người dùng phê duyệt] |
| B-7 | Hook endpoint trên listener riêng | Giảm bề mặt tấn công | [Đã xác minh] SECURITY §7 |
| B-8 | pino + correlationId | Truy vết xuyên API/worker/event | [Đề xuất] |

## 5. Dependencies

- Gói chính: `@nestjs/*`, `@nestjs/bullmq`, `bullmq`, `drizzle-orm`/`drizzle-kit`, `postgres`, `ioredis`, `socket.io` + `@socket.io/redis-adapter`, `zod`, `nestjs-pino`, `dockerode`, `xstate` v5, `@anthropic-ai/claude-agent-sdk`, `@napi-rs/keyring`, `tree-kill`, `uuidv7` (tên và phiên bản cuối chốt ở Phase 1).
- Hạ tầng: Postgres 16, Redis 7, Docker Engine, Git ≥ 2.x với worktree, Claude Code CLI, Codex CLI (chưa cài).
- Tài liệu: `DATABASE_DESIGN.md`, `SECURITY.md`, `ORCHESTRATION_WORKFLOW_RESEARCH.md`, `AGENT_SCHEDULER.md`, `WORKSPACE_AND_GIT_ISOLATION.md`, `INFRASTRUCTURE_DESIGN.md`.

## 6. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Semaphore leak khi worker chết | lease TTL + `presence-reaper` đối chiếu (ORCHESTRATION §10) |
| Hai scheduler tick cùng run | `SELECT FOR UPDATE` + jobId dedupe |
| Outbox poll làm tăng tải/độ trễ | index partial `published=false`, batch 500; đo ở Phase 1 |
| `bigserial global_seq` không đảm bảo thứ tự commit | consumer chấp nhận out-of-order, dựa `sequence` per aggregate (DATABASE_DESIGN §5.1) |
| Zod → OpenAPI chưa phủ hết trường hợp | snapshot test OpenAPI, bổ sung annotation tay |
| Drizzle relational API non-mature | dùng explicit join |
| Windows kill child tree | `tree-kill`/`taskkill`, test riêng Win11 |
| Output agent lớn làm phình DB | ghi artifact, `output` ≤ 32 KB (ORCHESTRATION §10) |
| Nhiều module song song sửa cùng bảng | quy tắc sở hữu bảng §3.2 + review |

## 7. Vấn đề chưa giải quyết

1. Thêm `event_outbox` (45 bảng) và `event_sequences`? (DATABASE_DESIGN §12 mục 7)
2. pgvector cho `agent_memories.embedding` ở v1 hay v2?
3. RLS từ v1 hay chỉ app-level scoping?
4. Mô hình enum trạng thái: `task_status` UPPERCASE hay lowercase (DECISION_LOG mục 5)?
5. Enum `agent_status` 12 trạng thái (PRD) và phép chiếu sang trạng thái office (idle/working/...): cần xác nhận với Database agent.
6. Chia `api` và `worker` thành hai process ở desktop hay in-process?
7. Hook listener cổng riêng khi chạy trong Tauri sidecar: cấp port thế nào?
8. Tên chuẩn của provider kind (`claude_code` vs `claude-code`).

## 8. Acceptance Criteria

- [ ] Mỗi module §3.2 có thư mục, `*.module.ts`, README ngắn nêu bảng sở hữu và API export.
- [ ] Lint chặn import chéo repository giữa module.
- [ ] Mọi endpoint mutation đi qua guard auth + `TenantContext` + `AuditInterceptor`; test chứng minh không rò tenant.
- [ ] Cùng một lỗi domain cho ra cùng envelope ở REST và WS ack.
- [ ] Mutation tạo `agent_events` và `event_outbox` trong cùng transaction (test rollback không để sót event).
- [ ] Config sai làm process thoát khi boot với thông báo rõ.
- [ ] Log không chứa token/secret (test redaction).
- [ ] Có `FakeAgentAdapter` và test Conductor chạy xong template SDLC không cần CLI thật.
- [ ] Chaos test: kill worker giữa node, run phục hồi mà không rò semaphore.
