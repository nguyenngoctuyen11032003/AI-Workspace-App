# API_DOCUMENTATION — REST, WebSocket và hợp đồng dữ liệu

> Tác giả: System Architect · Phase 0 (chỉ tài liệu) · Draft v0.1 · 2026-10-30
> Nhãn: **[Đã xác minh]** (có nguồn) · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**. Các D-xxx đều "Chờ duyệt" (`DECISION_LOG.md`).
> **Toàn bộ endpoint trong tài liệu này là [Đề xuất]**: chưa có tài liệu nguồn nào định nghĩa REST path; chúng được suy ra từ các module (`BACKEND_ARCHITECTURE.md` §3.2), bảng DB (`DATABASE_DESIGN.md` §4), route UI (`NAVIGATION_STRUCTURE.md` §4.1, §7) và yêu cầu PRD (§5). Tên nguồn cho từng tài nguyên có ghi ở mỗi bảng.

---

## 1. Mục đích

Cung cấp bản hợp đồng API đủ chi tiết để FE và BE phát triển song song: quy ước chung, bảng tài nguyên REST theo module, kênh WebSocket/Socket.IO, event envelope, kế hoạch sinh OpenAPI và quy trình "contract first".

## 2. Phạm vi

- **Trong:** quy ước (versioning, pagination, error, idempotency, auth), tài nguyên REST của mọi module, kênh realtime, event envelope, OpenAPI plan.
- **Ngoài:** schema bảng, logic nghiệp vụ, thiết kế màn hình. Payload chi tiết từng DTO là việc của `packages/contracts` (Phase 1), tài liệu này chỉ cố định hình dạng và tên.

## 3. Quy ước chung

### 3.1 Base URL, versioning, định dạng

- Base: `http://127.0.0.1:<port>/api/v1` (desktop; port ngẫu nhiên ghi ở `%LOCALAPPDATA%\aiw\run\server.json`, `docs/SECURITY.md` §7 **[Đã xác minh]**). Server mode: `https://<host>/api/v1`.
- **Versioning [Đề xuất]:** version trong path (`/api/v1`). Thay đổi tương thích ngược (thêm trường tùy chọn, thêm endpoint) không tăng version; thay đổi phá vỡ sẽ tạo `/api/v2` và giữ `v1` tối thiểu một phase. Header `X-API-Version` trả ngược để chẩn đoán.
- JSON `camelCase`, `Content-Type: application/json`, UTF-8; thời gian ISO-8601 UTC; ID là chuỗi uuid v7 (DATABASE_DESIGN §0 **[Đã xác minh]**); tiền tệ là chuỗi decimal USD; body ≤ 1 MB (SECURITY §7).
- Mọi response có header `X-Correlation-Id` (khớp `correlationId` trong lỗi và log).
- Tài nguyên có `deletedAt` (soft delete) không lộ ra; `DELETE` trả `204`. Xóa agent là soft delete (DATABASE_DESIGN §7).

### 3.2 Error envelope

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Dữ liệu không hợp lệ",
    "status": 422,
    "details": [{ "path": "model", "issue": "required" }],
    "correlationId": "01J9ZQ...",
    "retryAfterMs": null
  }
}
```

| HTTP | `code` (enum trong `@aiw/contracts`) | Ý nghĩa |
|---|---|---|
| 400 | `BAD_REQUEST` | cú pháp sai |
| 401 | `UNAUTHENTICATED`, `TOKEN_EXPIRED` | thiếu/sai bearer |
| 403 | `FORBIDDEN`, `POLICY_DENIED`, `ORIGIN_NOT_ALLOWED` | không đủ quyền hoặc chính sách |
| 404 | `NOT_FOUND` | không có (hoặc thuộc org khác: không phân biệt để tránh dò) |
| 409 | `CONFLICT`, `INVALID_STATE_TRANSITION`, `IDEMPOTENCY_KEY_REUSED`, `APPROVAL_ALREADY_DECIDED`, `FILE_OWNERSHIP_VIOLATION` | xung đột trạng thái |
| 412 | `PRECONDITION_FAILED` | `If-Match` không khớp `version` |
| 422 | `VALIDATION_FAILED` | Zod lỗi |
| 429 | `RATE_LIMITED`, `PROVIDER_RATE_LIMITED`, `BUDGET_EXCEEDED` | giới hạn; có `retryAfterMs` |
| 502/503 | `PROVIDER_UNAVAILABLE`, `AUTH_EXPIRED`, `SANDBOX_UNAVAILABLE` | phụ thuộc ngoài |
| 500 | `INTERNAL` | không lộ stack (SECURITY §7) |

`message` được i18n phía FE theo `code`; backend trả tiếng Anh ổn định để log. Lỗi WS ack dùng cùng envelope.

### 3.3 Pagination, lọc, sắp xếp

- **Cursor-based** cho danh sách lớn hoặc theo thời gian (events, activity, audit, usage records, messages): `?limit=50&cursor=<opaque>` (mặc định 50, tối đa 200). Response: `{ "items": [...], "nextCursor": "..."|null }`.
- **Offset** chỉ cho danh sách nhỏ quản trị (skills, departments): `?page=1&pageSize=25`, response thêm `total`. **[Đề xuất]**
- Event theo session dùng `afterSeq` (DATABASE_DESIGN §5.4 **[Đã xác minh]**): `GET /sessions/:id/events?afterSeq=0&limit=500`.
- Lọc: query phẳng (`?status=running&departmentId=...&projectId=...`), nhiều giá trị bằng dấu phẩy. Sắp xếp: `?sort=-createdAt,name`. Tìm kiếm: `?q=`.
- Các filter khớp query param của UI (`?project=`, `?from=&to=&groupBy=` ở NAV §7).

### 3.4 Idempotency và đồng thời

- Mọi `POST` tạo tài nguyên hoặc kích hoạt hành động (run, approve, spawn) chấp nhận header `Idempotency-Key` (chuỗi ≤ 64 ký tự). Backend lưu `aiw:idemp:{key}` TTL 24 h (DATABASE_DESIGN §10 **[Đã xác minh]**); lặp lại cùng key + cùng body trả lại kết quả cũ; cùng key khác body trả `409 IDEMPOTENCY_KEY_REUSED`.
- `PATCH`/`PUT` dùng `If-Match: <version>` (cột `version` hoặc `updatedAt`) để chống ghi đè; sai trả `412`. **[Đề xuất]**
- Quyết định approval là idempotent theo `approvalId`: lần thứ hai trả `409 APPROVAL_ALREADY_DECIDED` kèm quyết định hiện tại.
- Hành động dài (start run, nghiệm thu) trả `202 Accepted` + `{ "runId": ... }`; tiến trình theo dõi bằng WS.

### 3.5 Xác thực và phân quyền

- **Desktop [Đã xác minh, SECURITY §7]:** `Authorization: Bearer <token 32 byte per-launch>`; WS dùng subprotocol `aiw-token.<token>` hoặc first-message trong 2 s; không đặt token vào query string; không cookie nên không CSRF; Origin allowlist; rate limit 100 req/10 s/token; 3 lần auth sai thì ngắt WS.
- **Server mode [Giả định, DATABASE_DESIGN §8 nêu JWT/session]:** JWT/session do `auth` cấp; `organization_id` từ token đặt vào `TenantContext`.
- Endpoint `/hooks/*` ở **listener riêng**, chỉ nhận `AIW_HOOK_TOKEN`, không nhận token UI (SECURITY §7).
- Ký hiệu cột `Auth` trong bảng dưới: `B` = bearer hợp lệ (mọi vai trò), `B:W` = cần quyền ghi (Owner/Director; MVP chỉ có một người dùng nên tương đương `B`), `B:A` = quyền quản trị/nguy hiểm (kèm `ConfirmDialog` phía UI), `H` = hook token, `P` = public (chỉ health). Vai trò tương lai Project Owner/Viewer: PRD §4 (**Giả định**, chi tiết ở `docs/product/USER_ROLES_AND_PERMISSIONS.md`, chưa đối chiếu trong tài liệu này).

## 4. Tài nguyên REST theo module

Tiền tố `/api/v1` được lược bỏ. Cột "Nguồn" cho biết tài liệu hậu thuẫn nhu cầu.

### 4.1 `auth`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/health` | liveness | P |
| GET | `/auth/session` | user, org, vai trò hiện tại, `appMode` | B |
| POST | `/auth/login` | chỉ server mode; trả JWT | P |
| POST | `/auth/logout` | thu hồi phiên | B |
| POST | `/auth/token/rotate` | đổi bearer token (desktop) | B:A |
| GET | `/auth/members` ; POST/PATCH/DELETE `/auth/members/:id` | quản lý thành viên (server mode) | B:A |

### 4.2 `organizations`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/organizations` ; GET `/organizations/:id` | liệt kê/xem (MVP: 1 org `local`) | B |
| POST | `/organizations` | tạo công ty (multi-company: Phase 2, PRD §5.1) | B:A |
| PATCH | `/organizations/:id` | sửa tên, cấu hình | B:W |
| DELETE | `/organizations/:id` | soft delete | B:A |
| GET | `/organizations/:id/seed-status` ; POST `/organizations/:id/seed` | seed 12 phòng ban + 18 vai trò (D-016) idempotent | B:A |

### 4.3 `departments`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/departments` ; GET `/departments/:id` | danh sách/chi tiết | B |
| POST | `/departments` | thêm phòng ban tùy ý (Mobile, SEO...) | B:W |
| PATCH | `/departments/:id` | sửa, đặt head, ngân sách | B:W |
| DELETE | `/departments/:id` | soft delete (từ chối nếu còn agent hoạt động) | B:A |
| GET | `/org-chart` | cây phòng ban + agent + quan hệ báo cáo | B |
| PUT | `/org-chart/layout` | lưu kéo-thả | B:W |

### 4.4 `agents`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/agents` | danh sách (`?status=&departmentId=&providerId=&q=`) | B |
| POST | `/agents` | Agent Builder | B:W |
| GET | `/agents/:id` | profile | B |
| PATCH | `/agents/:id` | cập nhật | B:W |
| DELETE | `/agents/:id` | soft delete | B:A |
| POST | `/agents/:id/pause` ; `/resume` ; `/disable` ; `/enable` | điều khiển lifecycle | B:W |
| POST | `/agents/:id/stop` | dừng cưỡng bức | B:A |
| POST | `/agents/emergency-stop` | dừng tất cả, thu hồi cached approval (SECURITY §5.3) | B:A |
| GET | `/agents/:id/{tasks,activity,conversations,usage,performance,errors}` | các tab profile | B |
| PUT/DELETE | `/agents/:id/skills/:skillId` | gán/gỡ skill | B:W |
| GET/POST | `/agents/:id/memories` ; PATCH/DELETE `/agents/:id/memories/:memId` | memory có luồng duyệt (D-013) | B:W |
| POST | `/agents/:id/messages` | chat trực tiếp với agent | B:W |
| GET | `/agents/:id/permissions` ; PUT | mức quyền | B:A |

### 4.5 `skills`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/skills` | Library (`?category=&q=`) | B |
| POST | `/skills` | tạo custom skill | B:W |
| GET | `/skills/:id` ; PATCH ; DELETE | CRUD | B / B:W / B:A |
| GET/POST | `/skills/:id/versions` | version bất biến | B / B:W |
| POST | `/skills/:id/documents` (multipart) ; DELETE `/skills/:id/documents/:docId` | upload MD/PDF/references | B:W |
| POST | `/skills/teach` | Teach Agent: parse chỉ dẫn thành đề xuất skill (chưa lưu) | B:W |
| POST | `/skills/:id/verify` | kiểm tra agent đã nạp | B:W |
| POST | `/skills/:id/evaluate` | chạy evaluation | B:W |
| GET | `/skills/:id/agents` | agent đang dùng | B |

### 4.6 `providers`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/providers` ; GET `/providers/:id` | trạng thái, model, usage | B |
| POST | `/providers` | thêm kết nối (claude_code, codex, openhands, generic_cli) | B:W |
| POST | `/providers/:id/connect` ; `/disconnect` ; `/authenticate` | trả hướng dẫn nếu chưa cài/đăng nhập; không nhận mật khẩu/token | B:W |
| POST | `/providers/:id/check` | CheckConnection (healthcheck CLI, `claude auth status`) | B |
| GET | `/providers/:id/models` | ListModels | B |
| GET | `/providers/:id/usage` | GetUsage | B |
| GET | `/providers/:id/errors` | GetError | B |
| PATCH | `/providers/:id` | `maxConcurrent`, pricing | B:A |
| DELETE | `/providers/:id` | gỡ | B:A |

Nguồn: PRD §5.4 (danh sách thao tác chuẩn). **[Đã xác minh]** luật: không hỏi mật khẩu, không trích token, không trả credential lên FE.

### 4.7 `projects`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/projects` | danh sách (`?status=active`) | B |
| POST | `/projects` | tạo từ prompt; tùy chọn `startWorkflow: true` | B:W |
| GET | `/projects/:id` ; PATCH ; DELETE | CRUD | B / B:W / B:A |
| GET | `/projects/:id/revisions` | phiên bản/revision | B |
| GET | `/projects/:id/preview` | URL/trạng thái preview | B |
| POST | `/projects/:id/preview/start` ; `/stop` | khởi/dừng preview (không production) | B:W |
| POST | `/projects/:id/acceptance` | nghiệm thu `{decision: approve|reject, feedback, markups[]}` | B:W |
| GET | `/projects/:id/artifacts` ; GET `/artifacts/:id` ; GET `/artifacts/:id/content` | artifact, log, báo cáo | B |

### 4.8 `tasks`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/projects/:projectId/tasks` | kanban/table/timeline (`?view` chỉ là UI) | B |
| POST | `/projects/:projectId/tasks` | tạo task | B:W |
| GET | `/tasks/:id` ; PATCH ; DELETE | CRUD | B / B:W / B:A |
| POST | `/tasks/:id/assign` | MANUAL hoặc `{mode:"auto"}` (PM scoring) | B:W |
| POST | `/tasks/:id/status` | đổi trạng thái hợp lệ | B:W |
| POST | `/tasks/:id/{retry,cancel,reassign}` | điều khiển | B:W |
| GET/POST/DELETE | `/tasks/:id/dependencies` | phụ thuộc | B:W |
| GET | `/tasks/:id/{history,events,diff}` | lịch sử gán, event, diff | B |

### 4.9 `orchestrator` (workflows, runs)
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET/POST | `/projects/:projectId/workflows` | danh sách/tạo (hoặc từ template) | B / B:W |
| GET | `/workflow-templates` ; GET `/workflow-templates/:id` | template mặc định "FULL SOFTWARE DEVELOPMENT LIFECYCLE" | B |
| GET/PATCH/DELETE | `/workflows/:id` | definition React Flow-compatible | B / B:W / B:A |
| POST | `/workflows/:id/versions` | publish version bất biến | B:W |
| POST | `/workflows/:id/validate` ; `/dry-run` | validate, dry-run | B:W |
| POST | `/workflows/:id/runs` | khởi chạy (202) | B:W |
| GET | `/runs` ; GET `/runs/:id` | danh sách/chi tiết, pin `definitionVersionId` | B |
| POST | `/runs/:id/{pause,resume,cancel}` | điều khiển | B:W |
| GET | `/runs/:id/nodes` ; GET `/runs/:id/nodes/:nodeRunId` | trạng thái node, attempts | B |
| POST | `/runs/:id/nodes/:nodeRunId/retry` | retry node | B:W |
| GET | `/runs/:id/checkpoints` | checkpoint/time-travel | B |
| POST | `/runs/:id/revisions` | tạo revision có mục tiêu từ reject | B:W |

### 4.10 `scheduler`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/scheduler/queue` | task ready/đang chờ, lý do chờ (`SEMAPHORE_WAIT`) | B |
| GET | `/scheduler/limits` ; PATCH | concurrency global/provider (mặc định 5, tối đa 18; D-010) | B / B:A |
| GET | `/scheduler/assignments/preview?taskId=` | điểm gán agent (role, skill, workload...) | B |

### 4.11 `sessions`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/sessions` | danh sách (`?agentId=&taskId=&status=`) | B |
| GET | `/sessions/:id` | chi tiết | B |
| GET | `/sessions/:id/events?afterSeq=&limit=` | replay (§3.3) | B |
| POST | `/sessions/:id/{interrupt,resume,close}` | điều khiển | B:W |
| POST | `/sessions/:id/messages` | gửi message/inject | B:W |
| GET | `/sessions/:id/output?offset=&length=` | log thô (artifact `log`) | B |

### 4.12 `workspace` (git)
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/projects/:projectId/repo` | repo, nhánh, remote | B |
| GET | `/projects/:projectId/branches` ; `/commits?branch=` | nhánh, commit | B |
| GET | `/projects/:projectId/tree?ref=&path=` ; `/file?ref=&path=` | duyệt file (Monaco) | B |
| GET | `/projects/:projectId/diff?base=&head=` | diff | B |
| GET | `/worktrees` ; GET `/worktrees/:id` | worktree theo task | B |
| GET | `/tasks/:id/manifest` | ownership manifest (chỉ đọc) | B |
| POST | `/tasks/:id/validate-diff` | chạy diff validation theo manifest | B:W |
| POST | `/projects/:projectId/merge` | merge `task → integration/<project>`; **không** có endpoint tự merge vào `main` (D-009) | B:A |
| POST | `/projects/:projectId/push` | đẩy remote (tùy chọn) | B:A |
| POST | `/worktrees/:id/cleanup` | dọn | B:A |

### 4.13 `sandbox`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| POST | `/sandbox/jobs` | tạo job build/test/lint/typecheck/preview (`SandboxJob`) | B:W |
| GET | `/sandbox/jobs` ; GET `/sandbox/jobs/:id` | trạng thái, `SandboxResult` | B |
| POST | `/sandbox/jobs/:id/cancel` | hủy | B:W |
| GET | `/sandbox/jobs/:id/logs` | log | B |
| GET | `/sandbox/status` | Docker khả dụng, image | B |

### 4.14 `approvals`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/approvals` | Approval Center (`?status=pending&projectId=`) | B |
| GET | `/approvals/:id` | chi tiết: diff/lệnh (redacted), `riskTags`, `inputHash` | B |
| POST | `/approvals/:id/approve` | `{scope: once\|task, note}` | B:W |
| POST | `/approvals/:id/reject` | `{reason, rejectItems[]}` | B:W |
| POST | `/approvals/:id/request-changes` | yêu cầu sửa | B:W |
| POST | `/approvals/:id/reveal-raw` | "show raw" có audit | B:A |
| GET/POST | `/approval-policies` ; PATCH/DELETE `/approval-policies/:id` | rule model (SECURITY §5.2) | B / B:A |
| DELETE | `/approvals/cached/:id` | thu hồi cached decision | B:A |
| POST | `/hooks/claude/pre-tool-use` (listener riêng) | hook giữ kết nối chờ quyết định | H |
| POST | `/hooks/codex/approval` (listener riêng) | cầu nối approval Codex | H |

### 4.15 `events`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/events?afterId=&types=&agentId=&taskId=&runId=` | timeline toàn cục theo `global_seq`, cursor | B |
| GET | `/events/:id` | một event | B |
| GET | `/activity` | Activity Center (notification + filter) | B |

### 4.16 `usage` (cost)
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/usage/summary?from=&to=&groupBy=agent\|department\|project\|provider\|model` | từ `usage_daily_rollups` | B |
| GET | `/usage/records?date=&cursor=` | drill-down 1 ngày | B |
| GET | `/usage/budget` | so với `resource_limits`, `departments.budget_*` | B |
| PUT | `/usage/budget/:scope/:id` | đặt ngân sách | B:A |
| GET | `/dashboard/overview` ; `/dashboard/{projects,agents,workflows,cost,quality,system}` | Dashboard (chỉ dữ liệu thật; PRD §5.7) | B |

### 4.17 `office`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/office/layouts` ; `/office/layouts/:id` | layout, phòng (14), furniture, desk | B |
| POST/PATCH/DELETE | `/office/layouts/:id/{rooms,furniture,desks}` | chỉnh layout (Phase 2) | B:W |
| GET | `/office/layouts/:id/snapshot` | trạng thái hiện tại (vị trí, animation) cho bootstrap | B |
| PUT | `/office/layouts/:id/desks/:deskId/assign` | gán desk | B:W |

Vị trí/animation không có endpoint ghi từ FE: do projector phát từ event thật (PRD §5.6 **[Đã xác minh]**).

### 4.18 `audit`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/audit-logs?actor=&entity=&from=&to=&cursor=` | tra cứu (append-only) | B:A |
| GET | `/audit-logs/:id` | chi tiết (before/after whitelisted) | B:A |
| POST | `/audit-logs/export` | xuất `.jsonl.gz` | B:A |

### 4.19 `notifications`, `settings`, `system`
| Method | Path | Mục đích | Auth |
|---|---|---|---|
| GET | `/notifications` ; `/notifications/unread-count` | chuông, badge | B |
| POST | `/notifications/:id/read` ; `/notifications/read-all` | đánh dấu đã đọc | B |
| GET | `/counts` | badge sidebar: approvals pending, agents blocked/failed, tasks blocked, providers degraded... (NAV §4.1) | B |
| GET/PATCH | `/settings` ; `/settings/:section` | general, appearance, notifications, office, shortcuts | B / B:W |
| GET/PATCH | `/system/limits` | `resource_limits` | B / B:A |
| GET | `/system/info` ; `/system/diagnostics` | phiên bản, Docker/Redis/Postgres, CLI | B |
| POST | `/system/emergency-stop` | alias của `/agents/emergency-stop` | B:A |

## 5. WebSocket / Socket.IO channels

**[Đã xác minh]** Socket.IO 4 cho gateway (D-014, LICENSE_AND_REUSE §5.4); kênh Redis `aiw:ch:*` (DATABASE_DESIGN §10); client reconnect gửi `lastEventId` (ORCHESTRATION §7.3). Mapping room/namespace dưới đây là **[Đề xuất]**.

- Namespace `/realtime`; handshake auth như §3.5. Client gửi `subscribe { room }` / `unsubscribe`, server trả ack `{ok, lastEventId}`.

| Room (client) | Redis channel nguồn | Nội dung | Ghi chú |
|---|---|---|---|
| `org:{orgId}` | `aiw:ch:org:{orgId}:events` | mọi event bền sau commit | Dashboard, Activity |
| `project:{projectId}` | lọc từ org stream | event của project | Tasks, Workspace |
| `run:{runId}` | lọc | `NODE_*`, `WORKFLOW_*`, `HUMAN_*` | Workflow Builder |
| `agent:{agentId}` | lọc | status, heartbeat, tool | Agent Detail |
| `session:{sessionId}` | `aiw:ch:session:{id}` | event của session, kể cả `output.chunk` ephemeral | Console |
| `task:{taskId}` | `aiw:ch:task:{id}` | status, messages | Task drawer |
| `approvals:{orgId}` | `aiw:ch:approvals:{orgId}` | `approval.requested/decided` | Badge, toast |
| `office:{layoutId}` | `aiw:ch:office:{layoutId}` | `office.moved/animation` (ephemeral, ≤ 10 tick/s) | Virtual Office |

Client → server (có ack, envelope lỗi §3.2): `agent.control { agentId, action: pause|resume|cancel|inject_message, payload }`, `session.control`, `ping`. Lệnh được chuyển tới `aiw:ch:control:agent:{agentId}`; mọi lệnh có `Idempotency-Key` trong `meta`.

Giới hạn **[Đã xác minh, SECURITY §7]**: message ≤ 256 KB; ngắt sau 3 lần auth sai.

## 6. Event envelope

### 6.1 Schema

Hợp nhất ORCHESTRATION §7.1 (`WorkspaceEvent`) và DATABASE_DESIGN §5.1 (cột `agent_events`) **[Đề xuất]**:

```ts
interface EventEnvelope<T extends string = string, P = unknown> {
  id: string;                 // uuid v7/ulid: cursor replay toàn cục (map từ global_seq khi cần)
  type: T;                    // 'session.started' | 'task.status_changed' | ... (dotted lowercase)
  v: 1;                       // schemaVersion của payload
  occurredAt: string;         // ISO UTC
  organizationId: string;     // thay cho `workspaceId` trong ORCHESTRATION
  aggregate: { type: 'session'|'task'|'run'|'agent'|'approval'|'project'; id: string; sequence: number };
  correlation: { correlationId: string; causationId?: string;
                 agentId?: string; taskId?: string; runId?: string; nodeRunId?: string; sessionId?: string };
  actor: { kind: 'system'|'agent'|'human'; id: string };
  ephemeral?: true;           // không ghi DB (output.chunk, office.animation)
  payload: P;                 // kiểm tra bằng Zod theo `type` trong @aiw/contracts
}
```

Quy tắc: `sequence` tăng liên tục theo aggregate (client phát hiện gap); payload ≤ 8 KB, output lớn tham chiếu `{artifactId, offset, length}` (DATABASE_DESIGN §5.1); không bao giờ chứa secret.

### 6.2 Danh mục event

**[Đã xác minh]** bộ tên từ DATABASE_DESIGN §5.2: `session.*`, `llm.*`, `tool.*`, `file.*`, `shell.exec`, `git.commit/push`, `task.*`, `run.*`, `approval.requested/decided/expired`, `agent.status_changed/message_sent/memory_proposed`, `office.moved/animation`. Bổ sung từ ORCHESTRATION §7.2 **[Đề xuất]** (đổi sang dotted lowercase): `agent.created/updated/deleted`, `agent.heartbeat`, `task.reassigned`, `assignment.blocked`, `workflow.started/paused/resumed/cancelled/completed/failed`, `workflow.stage_changed`, `node.ready/queued/started/succeeded/failed/skipped/retry_scheduled`, `human.approved/rejected` (tương đương `approval.decided` kèm `decision`), `revision.created`, `checkpoint.saved`, `semaphore.wait`, `worktree.created/merged/conflict`, `test.result`, `deployment.result`, `provider.status`, `cost.threshold`, `notification.created`, `entity.updated`.

### 6.3 Đối chiếu tên

**[OWR-2 Q-19 / D-014: ĐÃ DUYỆT quy ước dotted lowercase trong DB; UPPER_SNAKE chỉ là ánh xạ API qua MỘT hàm; replay bằng `lastEventId`; PENDING_OWNER_CLARIFICATION: Owner cần cung cấp/xác nhận mười sáu tên event gốc.]** Đặc tả "Prompt 1 mục 8.4" mà D-014 tham chiếu **không có trong thư mục `docs/`** nên không đối chiếu được (**Giả định**: tên UPPERCASE như `AGENT_CREATED ... HUMAN_REJECTED` ở PRD §5.5). Quy ước đề xuất: `type` trong envelope là dotted lowercase; tài liệu/UX gọi theo tên UPPERCASE bằng hàm map `toLegacyName()` một chiều. **[Cần người dùng phê duyệt]**.

Phép chiếu notification (NAV §10): `approval.requested` → category `approval.requested`; `agent.status_changed (blocked)` → `agent.blocked`; `workflow.*` → `workflow.run.*`; `provider.status`, `cost.threshold`, `system`.

### 6.4 Replay và thứ tự

1. Mở màn hình: REST snapshot rồi subscribe room.
2. Dedupe `sequence <= lastSeq`; gap thì REST `afterSeq`.
3. Reconnect < 10 phút: server đọc Redis Stream buffer (~2000), nếu đã trim thì đọc DB (DATABASE_DESIGN §5.4).
4. Ephemeral không replay; client xin snapshot (`/office/layouts/:id/snapshot`).
5. `global_seq` có thể không đúng thứ tự commit tuyệt đối: client chấp nhận out-of-order giữa aggregate khác nhau (DATABASE_DESIGN §5.1).

## 7. Bảo mật và giới hạn

- Rate limit: 100 req/10 s/token; WS 256 KB/message; body 1 MB (**[Đã xác minh]** SECURITY §7). Endpoint nặng (run, preview, export) có giới hạn riêng **[Đề xuất]**.
- Mọi mutation ghi `audit_logs` (không ghi `system_prompt` đầy đủ, chỉ hash + length: DATABASE_DESIGN §7).
- Response không chứa `credential_ref`, token CLI, hay nội dung chưa redact trừ `reveal-raw` có audit.
- CORS: không `*`, không bật credentials; Origin allowlist (SECURITY §7).

## 8. Endpoint nội bộ (không thuộc OpenAPI công khai)

- `/hooks/*` trên listener riêng (H); schema vẫn ở `@aiw/contracts` nhưng không xuất vào Swagger UI.
- Bridge dev: `/_dev/*` chỉ khi `NODE_ENV=development` **[Đề xuất]**.

## 9. Kế hoạch sinh OpenAPI

**[Đề xuất]**
1. Zod trong `packages/contracts` là nguồn duy nhất (request, response, event, error).
2. Backend dùng `nestjs-zod` (hoặc `@asteasolutions/zod-to-openapi`) để sinh `openapi.json` lúc build: `pnpm --filter @aiw/api openapi:generate` ghi `packages/contracts/openapi/openapi.json` và commit.
3. FE sinh client kiểu từ OpenAPI (`openapi-typescript`) hoặc dùng trực tiếp Zod; hook TanStack Query viết tay mỏng trên đó.
4. CI: so khớp `openapi.json` với snapshot (lệch thì fail); công cụ phát hiện breaking change (ví dụ `oasdiff`) chặn thay đổi phá vỡ trong cùng `v1`.
5. Event schema xuất thành JSON Schema/AsyncAPI (`asyncapi.json`) cho kênh WS **[Giả định]** chọn AsyncAPI; có thể chỉ giữ JSON Schema.
6. Swagger UI chỉ bật ở dev, bind localhost.
7. MSW mock FE sinh từ cùng schema.

## 10. Contract first: DTOs frozen before FE/BE parallel work

> **CẢNH BÁO QUY TRÌNH.** Không bắt đầu song song FE và BE cho một module khi DTO của module đó chưa "đóng băng".

**Quy trình [Đề xuất]:**
1. **Draft:** System Architect + BE + FE cùng viết Zod DTO (request, response, event payload, error codes) vào `packages/contracts` cho một module; kèm ví dụ JSON.
2. **Review:** FE xác nhận đủ cho màn hình tương ứng (`SCREEN_SPECIFICATION.md`), BE xác nhận khớp bảng (`DATABASE_DESIGN.md`).
3. **Freeze:** merge vào nhánh chính với tag `contracts/<module>@1`. Từ lúc này sửa phá vỡ cần RFC ngắn + bump version + cả hai bên đồng ý. Thêm trường tùy chọn thì được phép.
4. **Parallel:** BE hiện thực controller theo schema; FE dùng MSW mock sinh từ schema. Contract test chạy ở CI hai phía.
5. **Integrate:** thay mock bằng API thật; e2e xác nhận.

**Thứ tự đóng băng đề xuất:** (1) `errors`, `pagination`, `event envelope`; (2) `auth`, `organizations`, `departments`, `agents`; (3) `providers`, `sessions`, `events`; (4) `projects`, `tasks`, `orchestrator`, `approvals`; (5) `workspace`, `sandbox`, `usage`, `office`, `audit`, `skills`.

**Điều kiện freeze một module:** mọi endpoint trong §4 của module có Zod request/response; mọi event type liên quan có payload schema; mã lỗi dùng enum chung; ví dụ JSON hợp lệ qua schema; OpenAPI snapshot được tạo; mapping tới route/màn hình FE được ghi.

**Hạng mục chặn freeze (chưa giải quyết):** enum `task_status` hoa/thường (DECISION_LOG mục 5), enum `agent_status` 12 trạng thái, tên provider kind, bảng `event_outbox` (nay có đề xuất ở DATABASE_DESIGN §5.5, OWR-2 D-014), quyết định tên event §6.3 (đã duyệt hướng ở OWR-2 Q-19; `task_status` hoa/thường đã chốt ở Q-13: task_status UPPERCASE).

## 11. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| A-1 | Versioning theo path `/api/v1` | đơn giản, dễ proxy/cache | [Đề xuất] |
| A-2 | Error envelope một dạng, có `code` + `correlationId` | FE map i18n, dễ truy vết | [Đề xuất] |
| A-3 | Cursor cho event/log, offset cho danh sách nhỏ | ổn định khi dữ liệu thay đổi nhanh | [Đề xuất]; replay bằng `afterSeq` [Đã xác minh] |
| A-4 | `Idempotency-Key` TTL 24 h | retry an toàn, khớp Redis key đã thiết kế | [Đã xác minh] DATABASE_DESIGN §10 |
| A-5 | Không có endpoint ghi vị trí office | chuyển động chỉ từ event thật (D-011/PRD §5.6) | [Đã xác minh] |
| A-6 | Không có endpoint merge vào `main` | D-009 | [Đã xác minh] |
| A-7 | Zod là nguồn sinh OpenAPI + event schema | một schema cho BE/FE | [Đề xuất] |
| A-8 | Dotted lowercase cho `type` event; UPPER_SNAKE chỉ qua một hàm ánh xạ `toLegacyName()` | khớp DATABASE_DESIGN; OWR-2 Q-12/Q-19/D-014 | **Đã duyệt (OWR-2 Q-19)**; danh sách event vẫn là đề xuất, PENDING_OWNER_CLARIFICATION (mục 8.4 gốc thiếu) |
| A-9 | Contract-first, freeze trước parallel | giảm vỡ tích hợp | [Đề xuất] |

## 12. Dependencies

- `docs/DATABASE_DESIGN.md` (bảng, event, Redis key), `docs/SECURITY.md` §5, §7, §8 (approval, auth, audit), `docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md` (workflow/run/event), `docs/product/PRD.md` + `FUNCTIONAL_REQUIREMENTS.md` (FR-xxx), `docs/design/NAVIGATION_STRUCTURE.md` (route, notification, deep link).
- `packages/contracts` và `@aiw/workflow-schema` (Phase 1).
- Công cụ: Zod, `nestjs-zod` hoặc `zod-to-openapi`, `openapi-typescript`, MSW, `oasdiff`.

## 13. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Endpoint đề xuất lệch yêu cầu FR chi tiết (chưa đối chiếu từng FR-xxx) | review với `FUNCTIONAL_REQUIREMENTS.md` trước freeze |
| DTO đổi sau khi freeze | RFC + version + contract test |
| Lệch tên event giữa tài liệu | §6.3, một nguồn `@aiw/contracts` |
| Lệnh điều khiển qua WS bị replay | `Idempotency-Key` trong meta, ack |
| Bề mặt API lớn gây rò quyền | guard mặc định deny, test tenant |
| OpenAPI sinh từ Zod thiếu chi tiết | annotation bổ sung + snapshot review |
| Preview/export nặng làm nghẽn | `202` + job + rate limit riêng |

## 14. Vấn đề chưa giải quyết

1. Chọn hệ tên event cuối cùng (dotted lowercase vs UPPERCASE) và tìm lại "Prompt 1 mục 8.4" (không có trong `docs/`).
2. Vai trò/quyền chi tiết (Viewer thấy Cost/Providers?) cần đối chiếu `USER_ROLES_AND_PERMISSIONS.md`.
3. Một `/counts` gộp hay nhiều endpoint count cho sidebar badge?
4. Dùng AsyncAPI hay chỉ JSON Schema cho event?
5. Cursor opaque: dựa `global_seq` hay `(occurredAt, id)`?
6. Endpoint upload skill dùng multipart hay pre-signed (chỉ server mode)?
7. Có cần GraphQL/SSE cho một số luồng không? (Đề xuất: không ở MVP.)
8. Chính sách `If-Match`: dùng cột `version` hay `updatedAt`? (DB chưa có cột `version` đã xác minh.)

## 15. Acceptance Criteria

- [ ] Mỗi module trong `BACKEND_ARCHITECTURE.md` §3.2 có bảng REST ở §4 (kiểm bằng checklist).
- [ ] Mọi endpoint có Zod request/response trong `@aiw/contracts`; `openapi.json` sinh được và khớp snapshot.
- [ ] Error envelope giống nhau ở REST và WS ack (contract test).
- [ ] Danh sách phân trang theo §3.3; replay `afterSeq` không mất/trùng event.
- [ ] `POST` nhạy cảm tuân `Idempotency-Key` (test gửi hai lần).
- [ ] Không endpoint nào trả credential/secret; không có endpoint ghi vị trí office; không có endpoint merge vào `main`.
- [ ] Mỗi module có tag freeze `contracts/<module>@1` trước khi FE/BE chia việc song song.
- [ ] Mâu thuẫn tên event (§6.3) được chốt và ghi `DECISION_LOG.md`.
