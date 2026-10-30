# AGENT_ARCHITECTURE.md — Mô hình Agent: template, instance, lifecycle, session và một lượt chạy end-to-end

Tác giả: Multi-Agent Architect — Phase 0 (chỉ tài liệu, không có mã ứng dụng). Ngày: 2026-10-30. Trạng thái: DRAFT chờ owner phê duyệt.
Tổng hợp từ: `DECISION_LOG.md` (D-001..D-016), `architecture/AGENT_ORGANIZATION.md`, `architecture/AGENT_SCHEDULER.md`, `architecture/AGENT_COLLABORATION.md`, `architecture/RESOURCE_MANAGEMENT.md`, `architecture/SKILL_SYSTEM.md`, `DATABASE_DESIGN.md` (§3, §4.2, §4.4, §4.7-4.9, §5), `research/ORCHESTRATION_WORKFLOW_RESEARCH.md`, `product/FUNCTIONAL_REQUIREMENTS.md`.
Tài liệu anh em: `MULTI_AGENT_ORCHESTRATION.md`, `WORKFLOW_ENGINE.md`, `EVENT_SYSTEM.md`. Tài liệu này **không lặp lại** danh mục 18 role, bảng ownership glob và scoring (xem `AGENT_ORGANIZATION.md`, `AGENT_SCHEDULER.md`); chỉ chốt mô hình khái niệm, state machine và luồng chạy.

Nhãn: **[Đã xác minh]** (đo/đọc từ nguồn trong repo hoặc thực nghiệm đã ghi trong tài liệu nguồn) · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.

---

## 1. Mục đích

Chốt một mô hình agent thống nhất để Phase 1 hiện thực mà không phải đoán: (1) phân biệt **AgentTemplate** (vai trò) và **AgentInstance** (nhân viên AI); (2) ba loại identity; (3) **vòng đời 12 trạng thái** với bảng transition được phép; (4) profile, memory, permission, provider binding; (5) mô hình **session**; (6) một lượt chạy agent từ lúc scheduler chọn đến lúc task có bằng chứng hoàn thành.

## 2. Phạm vi

- Trong phạm vi: khái niệm, state machine, ánh xạ sang bảng DB hiện có, hợp đồng Provider Adapter ở mức interface, luồng chạy.
- Ngoài phạm vi: thuật toán scoring (`AGENT_SCHEDULER.md`), workflow DAG (`WORKFLOW_ENGINE.md`), phân vai Orchestrator/PM (`MULTI_AGENT_ORCHESTRATION.md`), schema event (`EVENT_SYSTEM.md`), sandbox (`SECURITY.md`), cơ chế spawn từng CLI (`CLAUDE_CODE_INTEGRATION.md`, `CODEX_INTEGRATION.md`).

## 3. Nội dung chi tiết

### 3.1 Template và Instance [Đề xuất, dựa trên D-016]

| Khái niệm | Là gì | Vòng đời | Lưu ở đâu |
|---|---|---|---|
| **AgentTemplate** (role) | Bản mô tả vai trò: system prompt skeleton, skills mặc định, provider policy, permission tối đa, ownership patterns, model tier. 18 role mặc định là **seed data**, không hardcode (D-016). | Có version; archive thay vì xóa; clone để sửa. | Xem §3.1.1 (khoảng trống schema). |
| **AgentInstance** (agent) | "Nhân viên" cụ thể sinh từ template: tên, avatar, provider binding, override prompt/skill/permission, giới hạn riêng, trạng thái. Là đối tượng được giao task và hiện trong Virtual Office. | `created → ... → disabled`, xem §3.3. | `agents` (+ `agent_profiles`, `agent_provider_connections`). |
| **Session** | Một lần chạy CLI/SDK của instance cho một task attempt. Sinh ra bằng chứng và event. | `starting → active → exited/crashed`, xem §3.8. | `agent_sessions`. |

Quan hệ: Template 1—N Instance 1—N Session. Instance **pin** `templateVersion`; đổi template không ảnh hưởng session đang chạy (đã đề xuất ở `AGENT_ORGANIZATION.md` §7.1). Không có `switch(role)` trong business logic: mọi hành vi lấy từ dữ liệu template.

#### 3.1.1 Khoảng trống schema giữa A13 và `DATABASE_DESIGN.md` [Cần người dùng phê duyệt]

`DATABASE_DESIGN.md` §4.2 chỉ có bảng `agents` với cờ `is_template`; **không có** `template_version`, `skills[]` kèm weight, `ownership_patterns`, `provider_policy` của template. `AGENT_ORGANIZATION.md` §7 lại mô tả AgentTemplate có version. Hai cách xử lý (chọn một, owner duyệt):

1. **(Đề xuất ưu tiên)** Thêm bảng `agent_templates` + `agent_template_versions` (migration mới, không sửa 44 bảng đã thiết kế); `agents.template_version_id` là FK. `agents.is_template` giữ cho dữ liệu seed cũ hoặc bỏ.
2. Giữ `agents.is_template=true` làm template, version bằng cách clone dòng (mất lịch sử version rõ ràng, khó pin).

Tài liệu này dùng cách 1 ở các mô tả bên dưới; nếu owner chọn cách 2 chỉ phải đổi tên cột tham chiếu.

### 3.2 Ba loại identity [Đã xác minh động cơ, Đề xuất thiết kế]

Giữ nguyên bảng tại `AGENT_ORGANIZATION.md` §8, chỉ tóm tắt ràng buộc cần cho kiến trúc:

| Identity | Khóa | Dùng cho | Ràng buộc |
|---|---|---|---|
| User Identity | `users.id` | Phê duyệt, audit "ai ra lệnh" | Chỉ người dùng; không bao giờ do agent giả mạo. |
| Agent Identity | `agents.id` | Giao task, message bus, ownership, thống kê, log | Không chứa secret; chỉ có `credentialRef` (con trỏ). |
| Provider Identity | `provider_configs.id` (+ credential trong OS keychain) | Spawn, quota, cost | CLI tự quản credential (D-008); app không đọc `.credentials.json`/`auth.json`. |

Audit bắt buộc ghi bộ ba `(userId, agentId, providerId)`. Một Provider Identity phục vụ nhiều agent: tài khoản claude.ai `team` hiện tại dùng chung cho tất cả [Đã xác minh qua `claude auth status`] nên quota và semaphore tính theo provider/credential, không theo agent.

### 3.3 Vòng đời 12 trạng thái [Đề xuất trên enum đã có]

Nguồn enum: `agent_status` trong `DATABASE_DESIGN.md` §3 (`created, idle, available, assigned, running, waiting, blocked, reviewing, completed, failed, paused, disabled`) [Đã xác minh: enum tồn tại]. Ngữ nghĩa chi tiết dưới đây là **đề xuất** (nguồn chỉ liệt kê tên).

| State | Ngữ nghĩa | Giữ slot concurrency? | Office (tham chiếu `EVENT_SYSTEM.md` §9) |
|---|---|---|---|
| `created` | Instance đã tạo, chưa xác minh provider/profile. | Không | Spawn effect, chưa có ghế |
| `idle` | Đã bật, không có task, **chưa đủ điều kiện nhận việc** (ví dụ provider rate-limited, hết budget ngày, host guard). | Không | Phòng Relax/Coffee (chỉ khi có event chuyển sang idle, FR-164) |
| `available` | Đã bật, không có task, **đủ điều kiện** nhận việc (provider ok, budget ok, còn slot nhóm). | Không | Ngồi tại bàn |
| `assigned` | Scheduler đã gán task và giữ lease; session sắp/đang khởi động (`SPAWNING`). | Có (từ lúc acquire) | Đi tới bàn |
| `running` | Session `active`, đang làm task. | Có | Typing/Reading/Thinking |
| `reviewing` | Biến thể của running cho task loại REVIEW (agent đóng vai reviewer). Chung transition ra với `running`. | Có | Reading |
| `waiting` | Session tạm ngừng **và sẽ tự tiếp tục** khi có điều kiện: chờ human approval, chờ trả lời message, chờ reset quota. | Không (process đã dừng/idle), vẫn giữ workspace lock | Waiting |
| `blocked` | Cần **can thiệp** (AUTH lỗi, vi phạm ownership lặp lại, vượt budget, EMERGENCY_STOP). | Không | Error/bubble cảnh báo |
| `completed` | Task vừa có **bằng chứng nghiệm thu** (xem `MULTI_AGENT_ORCHESTRATION.md` §8). Trạng thái ngắn, chuyển tiếp ngay. | Không | Celebrating |
| `failed` | Attempt thất bại (crash, timeout, thiếu evidence); chờ quyết định retry/reassign. Trạng thái ngắn. | Không | Error |
| `paused` | Operator tạm dừng; lưu `resume_to` (state trước đó) và `providerSessionId`. | Không | Waiting/đứng yên |
| `disabled` | Admin vô hiệu hóa; không nhận task. | Không | Ẩn hoặc mờ |

Phân biệt `idle` và `available` giải quyết OI-FR-05 (Idle vs Available) theo hướng: `available = idle + eligibility`. Eligibility tính lại mỗi tick scheduler và khi provider/budget đổi; đổi qua lại phải **debounce ≥ 5 s** để không phát AGENT_STATE_CHANGED liên tục [Đề xuất].

#### 3.3.1 Bảng transition được phép

Mọi transition thực hiện bằng một service duy nhất (`AgentStateService.transition(agentId, to, reason, causedBy)`) trong cùng transaction với bản ghi `agent_events` (outbox, xem `EVENT_SYSTEM.md` §7). Transition không có trong bảng → `IllegalTransition` (log + không đổi state).

| # | Từ | Đến | Trigger | Guard | Ghi chú |
|---|---|---|---|---|---|
| T01 | created | idle | Provider connection `ok` và profile hợp lệ | `agent_provider_connections.status='ok'` | Lần bật đầu |
| T02 | created | disabled | Admin tắt trước khi kích hoạt | — | |
| T03 | idle | available | Eligibility = true | provider pool không RATE_LIMITED/DOWN, budget ngày còn, host guard ok | Debounce 5 s |
| T04 | available | idle | Eligibility = false | như trên | Lý do ghi vào payload |
| T05 | available | assigned | Scheduler assign | Qua hard filter (`AGENT_SCHEDULER.md` §5.3) + acquire semaphore ok | Tạo `task_assignments` |
| T06 | assigned | running | Session `active` (nhận event đầu tiên từ provider) | Trong `spawn_timeout` 60 s | `agent_sessions.status='active'` |
| T07 | assigned | reviewing | Như T06 nhưng task kind=REVIEW | — | |
| T08 | assigned | available | Assignment bị hủy hoặc spawn lỗi có thể retry | Giải phóng semaphore | Task về `READY` |
| T09 | assigned | blocked | Spawn lỗi AUTH hoặc workspace conflict kéo dài | — | Cần operator |
| T10 | running / reviewing | waiting | Chờ approval, chờ message reply, hoặc rate-limit có thời điểm reset | Session được suspend an toàn | Tự tiếp tục được |
| T11 | waiting | running / reviewing | Điều kiện thỏa (approve, message, quota reset) | Re-acquire slot nếu đã nhả | Có thể phải `--resume` (xem §3.8) |
| T12 | running / reviewing | blocked | Lỗi cần can thiệp | `failureClass ∈ {AUTH, BUDGET_EXCEEDED, OWNERSHIP_VIOLATION}` hoặc EMERGENCY_STOP | FR-214: task RUNNING → BLOCKED |
| T13 | blocked | running | Operator xử lý xong, session resume | — | |
| T14 | blocked | available | Task được release/reassign/cancel | Nhả workspace lock | |
| T15 | blocked | failed | Operator đánh dấu fail hoặc hết retry | — | |
| T16 | running / reviewing | completed | `TASK_COMPLETED` do EvidenceService xác nhận | **Có acceptance evidence** | Không có evidence thì không đi tới đây |
| T17 | running / reviewing | failed | Crash, timeout, stale, hoặc thiếu evidence | `failureClass` ghi lại | |
| T18 | assigned / running / reviewing / waiting | paused | Operator pause | Lưu `resume_to` | FR-239 |
| T19 | paused | resume_to | Operator resume | `resume_to ∈ {assigned, running, reviewing, waiting}` | Nếu resume_to là assigned/running cần slot |
| T20 | completed | available | Hết cooldown, không có task kế | Eligibility true | |
| T21 | completed | assigned | Scheduler có task kế ngay | Qua guard của T05 | Tránh flap |
| T22 | completed | idle | Hết cooldown, eligibility false | — | |
| T23 | failed | available | Quyết định retry/reassign xong, agent khỏe | `recentFailRate` dưới ngưỡng | Task đã quay về `READY` |
| T24 | failed | blocked | Lỗi mức agent (AUTH, crash lặp) | — | |
| T25 | mọi state trừ disabled | disabled | Admin vô hiệu hóa | Dừng session trước; task đang giữ về `READY`, ghi `reassign` | Có audit |
| T26 | disabled | idle | Admin bật lại | — | Quay lại T03 |

Không được phép (ví dụ điển hình): `created → running`, `idle → assigned` (phải qua `available`), `completed → running`, `disabled → bất kỳ state nào ngoài idle`, `waiting → completed` (phải về running để có evidence). Xóa instance là **soft delete** từ `disabled` (cột `deleted_at`).

#### 3.3.2 Ánh xạ ba state machine

| Agent state | Task status (`task_status`) | Session status (`session_status`) | Node run (`WORKFLOW_ENGINE.md` §5) |
|---|---|---|---|
| available | READY (chưa gán) | — | READY |
| assigned | ASSIGNED | starting | QUEUED |
| running | RUNNING | active | RUNNING |
| reviewing | REVIEWING (task review) | active | RUNNING |
| waiting (approval) | WAITING_APPROVAL | suspended | WAITING_HUMAN |
| waiting (quota/message) | RUNNING hoặc BLOCKED(reason) | suspended | RUNNING |
| blocked | BLOCKED | suspended / crashed | RUNNING hoặc FAILED |
| completed | COMPLETED | exited | SUCCEEDED |
| failed | FAILED hoặc READY (retry) | crashed / exited | FAILED |
| paused | giữ nguyên | suspended | giữ nguyên |

### 3.4 Profile [Đã xác minh schema, Đề xuất cách dùng]

`agent_profiles` (1-1 với `agents`): `personality`, `communicationStyle`, `strengths[]`, `weaknesses[]`, `preferredTools[]`, `bio`, `spriteConfig`, `stats` (cache: tasksDone, successRate, avgMinutes theo complexity).

- Profile phục vụ **hiển thị** (popover Virtual Office, trang agent) và **tín hiệu mềm** cho scoring (`stats.successRate`, `avgMinutes`). Không phải nguồn quyền hạn.
- `personality`/`communicationStyle` chỉ được nối vào system prompt khi owner bật; mặc định không, để giảm token và tránh hành vi không đoán trước [Đề xuất].
- `stats` là **cache có thể rebuild** từ `task_assignments` + `agent_sessions` + `usage_records`; không phải nguồn sự thật.

### 3.5 Memory [Đã xác minh schema và nguyên tắc; Đề xuất luồng]

Nguồn: `SKILL_SYSTEM.md` (memory tách khỏi skill, luôn qua phê duyệt, không tự lưu từ hội thoại; D-013), `agent_memories` (`type`: fact/preference/procedure/episode/summary; `scope`: agent/project/department/organization; `isApproved`).

1. Agent **đề xuất** ghi nhớ qua structured output `remember[]`; hệ thống tạo `agent_memories` với `isApproved=false` và phát `AGENT_MEMORY_PROPOSED`.
2. Chỉ memory `isApproved=true` mới được inject (digest ≤ 1 000 token khi spawn, theo `AGENT_COLLABORATION.md` §6). Người duyệt: owner/admin; có thể gom duyệt theo lô.
3. Giới hạn 200 entry/agent; vượt ngưỡng tạo task tóm tắt (role `docs` hoặc chính agent).
4. Memory **không** chứa secret hoặc nội dung nhạy cảm đã bị redact (`EVENT_SYSTEM.md` §8). Embedding `pgvector` là tùy chọn [CẦN DUYỆT ở DATABASE_DESIGN]; MVP dùng full-text search.
5. Context đã inject được **snapshot** vào `task_context_snapshot` để tái lập "agent đã biết gì" (evidence cho post-mortem).

### 3.5.1 Skill gắn vào instance [Đề xuất]

Skill theo chuẩn Agent Skills, nạp qua `--add-dir` (Claude) hoặc junction `.agents/skills` (Codex) theo D-013. Instance có `extraSkills`/`removedSkills` so với template; tập skill cuối cùng được **materialize** vào workspace task ở bước 5 của §3.9 và hash vào `task_context_snapshot`.

### 3.6 Permission [Đề xuất] — [Cần người dùng phê duyệt]

Hai nguồn chưa khớp (xem §8): `AGENT_ORGANIZATION.md` có 5 mức L0..L4; `DATABASE_DESIGN.md` enum `permission_level` có 3 giá trị (`read_only, standard, elevated`; giá trị `full` đã bị xóa theo OWR-2 Q-03). Đề xuất ánh xạ:

| DB enum | Mức A13 | Quyền | Cơ chế thực thi (D-004, D-007, D-009) |
|---|---|---|---|
| read_only | L0 | Đọc workspace | `--permission-mode plan`/allowlist Read/Grep/Glob |
| standard | L1-L2 | Ghi file khớp ownership manifest; ghi code trong worktree riêng | PreToolUse hook kiểm glob; `canUseTool` từ chối ngoài manifest |
| elevated | L3 | Chạy test/build trong Docker | Bash allowlist theo lệnh; timeout per command |
| ~~full~~ (đã xóa) | L4 | Lệnh infra, git push/tag, deploy | **[OWR-2 Q-03: KHÔNG hỗ trợ quyền `full`; enum `permission_level` chỉ có `read_only, standard, elevated`]** L4 (và L5) đi qua `elevated` + `permission_profile` (cột mới ở DATABASE_DESIGN §4 agents); **mỗi lệnh L4 tạo approval** |

**[OWR-2 Q-03]** Owner chấp thuận permission level **L0 đến L5**, dùng `permission_level` + `permission_profile`. Xung đột định nghĩa còn mở (ghi ở DOC_SYNC_LOG_OWR2_B): `AGENT_ORGANIZATION.md` §6 định nghĩa L0..L4 (L1 = ghi docs, L2 = ghi code, L3 = chạy test/build, L4 = infra), còn `product/USER_ROLES_AND_PERMISSIONS.md` §4.2 định nghĩa L0..L5 (L1 = write-own, L2 = Developer, L3 = Lead/Reviewer, L4 = Operator, L5 = PM/Orchestrator). Chưa chọn bên nào là chuẩn; cần Owner/Architect chốt bảng L0..L5 duy nhất. Bảng ánh xạ enum ở trên là tạm thời.

Nguyên tắc: instance không vượt template; user hạ được per-instance/per-task; production deploy luôn cần approval không tắt được (FR-199). Hook là lớp **chính sách**, không phải ranh giới bảo mật; lệnh build/test/run chạy trong Docker (D-007). Docker daemon chưa chạy lúc kiểm tra nên mọi giả định về cgroup/ECI là [Giả định].

### 3.7 Provider binding [Đề xuất]

- `provider_configs` = tài khoản/kết nối (kind: `claude_code`, `openai` cho Codex, …; `maxConcurrent`, `credentialRef`).
- `agent_provider_connections` = binding của instance: `providerId`, `model`, `priority` (0 = primary), `overrides`, `status`.
- Template chỉ ghi **provider suggestion**; instance ghi **binding**. Provider là thuộc tính thay thế được (nguyên tắc 4 của `AGENT_ORGANIZATION.md`).
- **Fallback**: DB cho phép `priority ≥ 1`, trong khi `RESOURCE_MANAGEMENT.md` §9 quy định không auto-switch. Hợp nhất [Đề xuất]: các dòng `priority ≥ 1` là *alternate đã được phê duyệt*; chuyển sang alternate **chỉ khi có approval** hoặc cờ `auto_switch_allowed=true` ở cấp project (mặc định tắt).

Hợp đồng Provider Adapter (interface mức khái niệm; chi tiết ở tài liệu provider):

```ts
interface ProviderAdapter {
  kind: 'claude_code' | 'codex' | 'openhands';
  verifyAuth(conn: ProviderConn): Promise<AuthStatus>;                 // claude auth status; không đọc file credential
  start(spec: SessionSpec): Promise<SessionHandle>;                    // spawn/SDK query; trả providerSessionId khi có
  resume(handle: SessionHandle, input: ResumeInput): Promise<SessionHandle>;
  interrupt(handle: SessionHandle): Promise<void>;                     // mềm
  stop(handle: SessionHandle, mode: 'graceful' | 'kill_tree'): Promise<void>;
  events(handle: SessionHandle): AsyncIterable<ProviderEvent>;         // đã chuẩn hóa, xem EVENT_SYSTEM §6
  capabilities(): { resume: boolean; pause: boolean; streamUsage: boolean; permissionHook: boolean };
}
```

Tính năng provider không hỗ trợ trả `NOT_SUPPORTED` (FR-081 rủi ro) thay vì giả lập; scheduler dùng `capabilities()` để chọn pause = suspend hay pause = cancel + retry.

### 3.8 Session model [Đề xuất trên `agent_sessions`]

`AGENT_SCHEDULER.md` gọi là `AgentExecution` (8 state `SPAWNING..LOST`), DB gọi là `agent_sessions` (6 state). Hợp nhất: **một khái niệm, tên DB `agent_sessions`**, "execution" chỉ là từ đồng nghĩa trong tài liệu cũ.

| Scheduler (A13) | `session_status` (DB) | Cột phụ |
|---|---|---|
| SPAWNING | starting | — |
| RUNNING | active | `last_heartbeat_at` cập nhật |
| PAUSED | suspended | `resume_token` hoặc `provider_session_id` còn |
| (suspended, có thể resume) | resumable | `provider_session_id` đã xác minh dùng lại được |
| STOPPING | active + cờ `stopping` (trong `exit_reason`) | — |
| DONE / FAILED / KILLED | exited | `exit_code`, `exit_reason` (`ok`, `timeout`, `user_cancel`, `budget`, `evidence_missing`) |
| LOST / crash | crashed | `exit_reason='stale'`/`'crash'` |

Quy tắc:
- **1 session = 1 attempt = 1 worktree** (D-009: `agent/<slug>/<task-id>`); retry là session mới, không resume giữa chừng subprocess không idempotent (nguyên tắc 6 của A07 §2.1). Resume qua `providerSessionId` chỉ dùng cho **pause/approval/rate-limit**, và là [Giả định] cho tới khi A05/A06 kiểm thử `--resume` với `-p`.
- Mỗi session có `lastEventSeq` (cursor per-aggregate), `pid`, `cwd`, `worktreeId`, `providerSessionId` (evidence để trace, **không** phải identity).
- Heartbeat có **hai lớp**: lớp process (`last_heartbeat_at`, mỗi 30 s, stale 3 phút theo A07) và lớp tiến triển (event stdout/file change, soft 5 phút, hard 20 phút theo A13). Hai lớp trả lời hai câu hỏi khác nhau (sống? có tiến triển?), giữ cả hai (xem §8, mâu thuẫn M-05).
- Mỗi spawn gắn `AIW_EXECUTION_ID` (env) để chống spawn đôi và để nhận diện process sau crash (PID reuse trên Windows).

### 3.9 Một lượt chạy agent end-to-end [Đề xuất]

Giả định đầu vào: một `node_run` kiểu agent đã `READY` (xem `WORKFLOW_ENGINE.md` §6) và task tương ứng ở `READY`.

| Bước | Thành phần | Hành động | Event phát ra (`EVENT_SYSTEM.md` §5) |
|---|---|---|---|
| 1 | Scheduler | Tính ready-set, chọn agent (AUTO) hoặc dùng agent do owner chọn (MANUAL); hard filter: role, skill, permission, ownership, provider, budget, workload | `TASK_ASSIGNED` (kèm score breakdown) |
| 2 | Resource Manager | Acquire semaphore Lua đa tầng (org → provider → project → agent); hết slot thì task ở `READY`, ghi `queuedReason` | `RESOURCE_SEMAPHORE_WAIT` nếu hết slot |
| 3 | AgentStateService | `available → assigned`; tạo `agent_sessions(starting)` và `task_assignments` | `AGENT_STATE_CHANGED` |
| 4 | Git Service | Tạo worktree + branch `agent/<slug>/<task-id>` (D-009), ghi manifest ownership theo glob của task | `WORKTREE_CREATED` |
| 5 | Skill/Context Builder | Materialize skill vào workspace; dựng context (task, acceptance criteria, files kèm lý do, decisions, outputs upstream, inbox liên quan, memory digest) trong `context_budget_tokens` (mặc định 8 000); lưu `task_context_snapshot` | — |
| 6 | Provider Adapter | `start()` qua `@anthropic-ai/claude-agent-sdk` (D-004) hoặc `codex app-server` (D-005); cài `canUseTool` + PreToolUse hook | `AGENT_SESSION_STARTED`, `PROVIDER_SESSION_STARTED` |
| 7 | AgentStateService | Nhận event đầu tiên trong 60 s: `assigned → running`, `TASK_STARTED`; nếu không: spawn fail → T08/T09 | `AGENT_STATE_CHANGED`, `TASK_STARTED` |
| 8 | Adapter + Event pipeline | Stream event đã chuẩn hóa; mỗi tool call kiểm quyền; heartbeat cập nhật; usage ghi vào ledger; kiểm budget từng event (hard stop ở cấp task) | `PROVIDER_TOOL_STARTED/COMPLETED`, `PROVIDER_USAGE`, `AGENT_HEARTBEAT`, `TASK_PROGRESS` |
| 9 | Approval gate (nếu cần) | Lệnh L4 hoặc hành động theo policy: tạo `approvals`, `running → waiting`, nhả slot, chờ human | `HUMAN_APPROVAL_REQUESTED`; sau đó `HUMAN_APPROVED/REJECTED` |
| 10 | Adapter | Session kết thúc: thu `output` có cấu trúc (summary, files, decisions, blockers) qua `--json-schema`/`--output-schema` | `PROVIDER_SESSION_ENDED`, `AGENT_SESSION_EXITED` |
| 11 | Git Service | **Diff gate**: so diff với manifest ownership (D-009: kiểm diff sau cùng là cổng quyết định); file ngoài manifest → fail attempt | `WORKTREE_CONFLICT` hoặc không có |
| 12 | EvidenceService | Thu bằng chứng nghiệm thu theo loại node (test result, diff đạt manifest, review PASS độc lập, approval) và ghi liên kết `tasks.result.artifactIds` | `TASK_EVIDENCE_RECORDED` |
| 13 | TaskService | Đủ evidence: task `COMPLETED`, node_run `SUCCEEDED`, agent `completed`; thiếu: attempt `FAILED(EVIDENCE_MISSING)` → quyết định retry | `TASK_COMPLETED` hoặc `TASK_FAILED` |
| 14 | Resource Manager | Release semaphore + lease; flush usage; `completed → available` | `AGENT_STATE_CHANGED` |
| 15 | Scheduler | Phát tick cho các run còn node `READY` (bù slot vừa nhả) | `WORKFLOW_NODE_READY` cho node kế |

Điểm then chốt: bước 12-13 là nơi áp **quy tắc "không COMPLETED nếu thiếu acceptance evidence"**; session thoát với exit code 0 **không** đủ.

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| AD-01 | Template và Instance tách bảng; instance pin version | Đổi vai trò không phá session đang chạy; khớp D-016 | [Đề xuất] — [Cần người dùng phê duyệt] (§3.1.1) |
| AD-02 | 12 state agent theo enum DB; `idle` vs `available` phân biệt bằng eligibility | Không đổi enum; giải OI-FR-05 | [Đề xuất] |
| AD-03 | `waiting` (tự tiếp tục) khác `blocked` (cần can thiệp) | UI và scheduler cần xử lý khác nhau (slot, thông báo) | [Đề xuất] |
| AD-04 | `completed` chỉ đạt khi EvidenceService xác nhận | Yêu cầu owner; chặn "agent tự báo xong" | [Đề xuất] — [Cần người dùng phê duyệt] |
| AD-05 | Một khái niệm `agent_sessions`; 1 session = 1 attempt = 1 worktree | Retry sạch, log/context không trộn | [Đề xuất] |
| AD-06 | Permission 5 mức A13 ánh xạ vào 4 enum DB | Tránh đổi enum, giữ hook mức lệnh cho L4 | [Đề xuất] — [Cần người dùng phê duyệt] |
| AD-07 | Memory chỉ inject khi đã duyệt; không tự lưu từ hội thoại | D-013, `SKILL_SYSTEM.md` | [Đã xác minh] nguyên tắc nguồn |
| AD-08 | Provider fallback chỉ qua approval | `RESOURCE_MANAGEMENT.md` §9: tiền và nhất quán là quyết định owner | [Đề xuất] |
| AD-09 | Credential do CLI quản lý; app chỉ gọi `verifyAuth` | D-008 | [Đã xác minh] quyết định log |

## 5. Dependencies

- `DATABASE_DESIGN.md`: `agents`, `agent_profiles`, `agent_provider_connections`, `agent_sessions`, `agent_memories`, `approvals`, `task_assignments`; **thêm** bảng template (§3.1.1) và bảng evidence (xem `MULTI_AGENT_ORCHESTRATION.md` §8.5).
- `CLAUDE_CODE_INTEGRATION.md`, `CODEX_INTEGRATION.md`: `capabilities()`, resume, permission hook, usage event.
- `SECURITY.md`: enforce permission, Docker, R1-R6 (D-007 cần xác nhận).
- `WORKSPACE_AND_GIT_ISOLATION.md`: worktree, manifest, diff gate.
- `EVENT_SYSTEM.md`: toàn bộ event ở bảng §3.9 và transition §3.3.1.

## 6. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| 12 state quá nhiều, dễ lệch giữa agent/task/node/session | Bug đồng bộ, UI sai | Một service transition duy nhất; test property "ma trận ánh xạ §3.3.2 luôn nhất quán"; state chỉ được ghi qua service |
| Flap `idle ↔ available` | Spam event, office nhấp nháy | Debounce 5 s, chỉ phát khi đổi thật |
| Resume (`--resume` với `-p`) không hoạt động | Pause/approval phải cancel + chạy lại, mất tiến độ | Đo ở Phase 1 spike; `capabilities().resume=false` thì pause = cancel + retry có context snapshot |
| Spawn tax ~48k token/lần (đo được) | Chi phí và quota | Config tối giản (`RESOURCE_MANAGEMENT.md` §4.3), gom task nhỏ |
| Schema thiếu bảng template/evidence | Phase 1 chặn | Chốt §3.1.1 trước khi viết migration |
| Hook là lớp chính sách, không phải sandbox | Agent vượt quyền nếu chỉ dựa hook | D-007: build/test/run trong Docker; diff gate sau cùng |
| 18 agent trên cùng quota subscription | Quota cạn | Ngân sách + semaphore theo provider; default 5 (D-010) |

## 7. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Cách lưu template (§3.1.1).
2. [Cần người dùng phê duyệt] Ánh xạ permission 5→4 (§3.6) và việc có cho template L4 do user tạo hay chỉ seed.
3. `completed` và `failed` là state "ngắn": owner có muốn chúng hiện ở UI/Office thời gian cố định (ví dụ 3 s) hay chỉ là transient trong event? [Đề xuất: chỉ transient trong DB, UI tự giữ hiệu ứng].
4. Cơ chế pause/resume thật của CLI trên Windows chưa kiểm [Giả định].
5. `agent_status` ghi ở `agents.status` (có `statusChangedAt`) và `agents.currentTaskId`: cần chốt thứ tự cập nhật với `tasks.status` để tránh hai nguồn sự thật (đề xuất: `tasks` là nguồn, `agents.currentTaskId` là cache cập nhật trong cùng transaction).
6. Codex CLI chưa cài [Đã xác minh] nên mọi hành vi Codex trong tài liệu này là [Giả định].

## 8. Mâu thuẫn giữa các tài liệu nguồn (liên quan tài liệu này)

| ID | Mâu thuẫn | Cách xử lý trong tài liệu này |
|---|---|---|
| M-01 | A07 định nghĩa agent status 5 giá trị (`idle/working/waiting/error/offline`); DB có 12 | Dùng 12 của DB; A07 hết hiệu lực ở điểm này |
| M-02 | A13 `AgentTemplate` có version; DB chỉ có `agents.is_template` | §3.1.1 |
| M-03 | A13 permission L0..L4 (5) vs DB `permission_level` (4) | §3.6 ánh xạ |
| M-04 | A13 `AgentExecution` (8 state) vs DB `agent_sessions` (6 state) | §3.8 hợp nhất |
| M-05 | A07 stale 3 phút (heartbeat process) vs A13 stale 5/20 phút (tiến triển) | Giữ hai lớp, khác ý nghĩa |
| M-06 | DB cho `agent_provider_connections.priority` fallback vs A13 "không auto-failover" | AD-08 |
| M-07 | Mặc định `provider_configs.max_concurrent=2` (DB) vs A13 claude 5, codex 3 | Cấu hình `ResourceConfig` thắng (D-010 APPROVED: Claude 5, Codex 3); DB default `max_concurrent` đã đổi từ 2 thành 5 |
| M-08 | A13 `maxRetries=3`, A07 `maxAttempts=3`, DB `agents.max_retry=2` / `tasks.max_retry=2` | Giá trị cuối lấy từ task (`tasks.max_retry`), mặc định cấu hình chung; cần owner chốt 2 hay 3 |

## 9. Acceptance Criteria (cho Phase 1)

- [ ] Có test ma trận T01-T26: mọi cặp state không có trong bảng ném `IllegalTransition`; mọi transition hợp lệ ghi đúng một `AGENT_STATE_CHANGED` (cùng transaction với thay đổi `agents.status`).
- [ ] Property test: với bất kỳ chuỗi event hợp lệ, ma trận §3.3.2 (agent/task/session/node_run) không bao giờ rơi vào tổ hợp không được liệt kê.
- [ ] Seed tạo đủ 18 template; thêm template thứ 19 không cần đổi mã (không có `switch(role)`; lint hoặc checklist review).
- [ ] Instance pin `templateVersion`: đổi template giữa session không đổi prompt/skill đã snapshot (integration test, sha khớp).
- [ ] Task không thể sang `COMPLETED` khi thiếu evidence (test ở TaskService và, nếu duyệt, trigger DB).
- [ ] 1 session = 1 attempt = 1 worktree; retry tạo session và branch mới; không có hai session `active` cho cùng instance khi `maxConcurrentExecutions=1`.
- [ ] Spawn đôi sau crash bị chặn nhờ `AIW_EXECUTION_ID` (test kill app giữa chừng).
- [ ] Memory chưa duyệt không xuất hiện trong context snapshot; memory đã duyệt xuất hiện tối đa 1 000 token.
- [ ] Audit log của mọi session có đủ `(userId, agentId, providerId)`.
- [ ] Owner đã duyệt các mục đánh dấu [Cần người dùng phê duyệt] ở §3.1.1, §3.6, AD-04.


> **Phụ lục sửa đổi (D-021, PROPOSED):** nạp skill Markdown động theo từng agent, quét lúc giao task, xem `docs/architecture/DYNAMIC_SKILL_INJECTION.md`. Phụ lục này bổ sung nguồn skill trên đĩa và rào chắn `.ai/**`; phần còn lại của tài liệu này giữ nguyên cho đến khi Owner duyệt.
