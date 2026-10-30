# EVENT_SYSTEM.md — Event system: envelope, danh mục event, outbox, replay, redaction, ánh xạ Virtual Office

Tác giả: Multi-Agent Architect — Phase 0 (chỉ tài liệu). Ngày: 2026-10-30. Trạng thái: DRAFT chờ owner phê duyệt.
Tổng hợp từ: `DECISION_LOG.md` (D-011, D-014), `research/ORCHESTRATION_WORKFLOW_RESEARCH.md` (A07 §7), `DATABASE_DESIGN.md` (§4.8, §4.12, §5, §10), `product/FUNCTIONAL_REQUIREMENTS.md` (FR-116..118, FR-158..165), `research/CHATDEV_ANALYSIS.md` (WS/event), `research/OPENHANDS_ANALYSIS.md` (§4.2, §10), `research/PIXEL_AGENTS_ANALYSIS.md` (§2.1, §2.3, §4.4), `architecture/AGENT_COLLABORATION.md`.
Anh em: `AGENT_ARCHITECTURE.md` (state, session), `MULTI_AGENT_ORCHESTRATION.md`, `WORKFLOW_ENGINE.md`. Số mục được giữ ổn định vì tài liệu anh em tham chiếu: §5 = danh mục, §6 = provider-normalized, §7 = outbox/Redis/replay, §8 = redaction, §9 = Virtual Office.

Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.

---

## 1. Mục đích

Định nghĩa **event** là nguồn sự thật duy nhất cho UI và Virtual Office (FR-118): schema envelope, danh mục event kèm payload, cách phát đáng tin cậy (outbox giao dịch → Redis pub/sub + Redis Stream, D-014), thứ tự, replay bằng `lastEventId`, redaction, retention, và bảng ánh xạ event → trạng thái/animation nhân vật. Quy tắc bất biến: **mọi chuyển động và animation trong Virtual Office chỉ do event thật điều khiển** (FR-160).

## 2. Phạm vi

- Trong: envelope, catalogue AGENT_/TASK_/WORKFLOW_/HUMAN_ và event khác, event provider chuẩn hóa, transport, ordering, replay, redaction, mapping office, retention.
- Ngoài: nội dung UI (`DESIGN_SYSTEM.md`), render Phaser (`PIXEL_AGENTS_ANALYSIS.md`, D-011), lược đồ SQL chi tiết (`DATABASE_DESIGN.md`), WebSocket gateway (Phase 1).

## 3. Nguyên tắc [Đề xuất, rút từ OpenHands và A07]

1. **Envelope tách khỏi payload**; payload có `type` phân biệt và `schemaVersion` (OpenHands §10 nguyên tắc 1). Mỗi `type` có zod schema trong `packages/contracts`.
2. **Ghi trạng thái và event trong một transaction** (outbox); không "emit rồi mới UPDATE" (A07 §2.1).
3. **Durable vs transient** tách ở tầng transport: durable có `globalSeq`/`sequence` và replay được; transient (delta, animation) chỉ qua Redis, không replay (OpenHands session frames).
4. **Ghép cặp** action/observation bằng `toolCallId` + `actionId` (OpenHands nguyên tắc 2); `llmResponseId` nhóm tool call song song.
5. **Envelope được phép lớn dần**: consumer gặp `type` lạ phải bỏ qua, không lỗi (OpenHands nguyên tắc 7).
6. **Không chứa secret, prompt thô, stdout dài** (xem §8).
7. UI và Virtual Office là **projection read-only** của event; không component nào cập nhật hiển thị ngoài event.

## 4. Envelope

### 4.1 Schema [Đề xuất, ánh xạ cột `agent_events`]

```ts
interface EventEnvelope<T extends string = string, P = unknown> {
  id: string;                 // UUID v7 (sắp xếp theo thời gian; cột uuid của DB) — xem M-E1
  globalSeq?: number;         // do DB cấp khi INSERT (bigserial); không có ở event transient
  schemaVersion: number;      // phiên bản payload của `type`
  type: T;                    // UPPER_SNAKE, ví dụ 'TASK_COMPLETED'
  occurredAt: string;         // ISO 8601 UTC; do server gán, không tin đồng hồ client/provider
  delivery: 'durable' | 'transient';
  organizationId: string;
  aggregate: { type: 'session'|'task'|'run'|'agent'|'approval'|'project'|'provider'|'system'; id: string };
  sequence?: number;          // per-aggregate, bắt đầu 1, liên tục (chỉ durable)
  refs: { projectId?: string; agentId?: string; taskId?: string; runId?: string; nodeRunId?: string; sessionId?: string };
  correlationId?: string;     // một yêu cầu/hành động xuyên nhiều aggregate
  causationId?: string;       // id của event trực tiếp gây ra event này
  actor: { kind: 'user'|'agent'|'system'|'provider'|'hook'; id?: string };
  payload: P;
  redactions?: string[];      // đường dẫn trường đã bị redact (xem §8)
}
```

### 4.2 Ánh xạ với `agent_events`

| Envelope | Cột DB | Ghi chú |
|---|---|---|
| `id` | `id` (uuid) | `gen_random_uuid()` mặc định là v4; dùng v7 để sắp xếp (cần hàm app hoặc PG 18); A07 dùng ULID |
| `globalSeq` | `global_seq` | cursor toàn cục |
| `aggregate.type/id`, `sequence` | `aggregate_type`, `aggregate_id`, `sequence` | unique `(type,id,sequence,occurred_at)` |
| `refs.*` | `agent_id, task_id, run_id, session_id, node_run_id` | denormalized để query mọi chiều |
| `correlationId`, `causationId` | cùng tên | |
| `actor` | `actor_user_id` + `payload.actor` | Cột chỉ có user; actor khác lưu trong envelope JSON |
| `schemaVersion` | `schema_version` | |
| `payload`, `occurredAt`, `type` | cùng tên | |
| `delivery`, `redactions` | **chưa có cột** | Transient không lưu; `redactions` đưa vào `payload._redactions` hoặc thêm cột (cần duyệt) |

Cấp `sequence` trong transaction: aggregate `session` dùng `UPDATE agent_sessions SET last_event_seq = last_event_seq + 1 RETURNING`; các aggregate khác dùng bảng nhỏ `event_sequences` (DB §5.1, đang [CẦN DUYỆT]).

## 5. Danh mục event

Quy ước: tên UPPER_SNAKE theo tiền tố; cột "D/T" = Durable (ghi `agent_events` + outbox) hoặc Transient (chỉ Redis). Trường `?` là tùy chọn. Mọi event có envelope §4; bảng chỉ liệt kê `payload`.

### 5.1 AGENT_*

| Type | Aggregate | D/T | Payload | Nguồn |
|---|---|---|---|---|
| AGENT_CREATED | agent | D | `agent{id,name,slug,templateCode,templateVersion,departmentId?,providerId?,permissionLevel}` | API |
| AGENT_UPDATED | agent | D | `agentId`, `changed[]` (tên trường, không giá trị nhạy cảm), `version?` | API |
| AGENT_ARCHIVED | agent | D | `agentId`, `reason?` (soft delete; thay `AGENT_DELETED` của A07) | API |
| AGENT_STATE_CHANGED | agent | D | `agentId`, `from`, `to` (12 state), `reason`, `taskId?`, `sessionId?`, `resumeTo?`, `failureClass?` | `AgentStateService` |
| AGENT_SESSION_STARTED | session | D | `sessionId`, `agentId`, `taskId`, `providerKind`, `model?`, `worktreeId?`, `attempt` | Adapter |
| AGENT_SESSION_SUSPENDED / _RESUMED | session | D | `sessionId`, `reason` (`pause`/`approval`/`rate_limit`), `providerSessionId?` | Adapter |
| AGENT_SESSION_EXITED | session | D | `sessionId`, `status` (`exited`/`crashed`), `exitCode?`, `exitReason`, `durationMs`, `tokens{in,out,cacheRead,cacheWrite}`, `costUsdList?` | Adapter/Reaper |
| AGENT_HEARTBEAT | session | T | `sessionId`, `lastLine?` (đã redact, ≤ 200 ký tự), `elapsedMs`, `tokensIn`, `tokensOut` | Worker (≤ 1 mỗi 5 s) |
| AGENT_MESSAGE_SENT | task/agent | D | `messageId`, `messageType`, `fromAgentId`, `to{kind,id}`, `threadId?`, `tokenEst`, `taskId?` (không chứa nội dung; nội dung ở `agent_messages`) | MessageBus |
| AGENT_MEMORY_PROPOSED | agent | D | `memoryId`, `agentId`, `scope`, `type`, `sourceTaskId?` | Orchestrator |
| AGENT_MEMORY_DECIDED | agent | D | `memoryId`, `approved`, `decidedBy` | API |

### 5.2 TASK_*

| Type | Aggregate | D/T | Payload |
|---|---|---|---|
| TASK_CREATED | task | D | `task{id,taskId,projectId,name,priority,parentTaskId?,nodeRunId?,requiredRole?}`, `createdBy{kind,id}` |
| TASK_ASSIGNED | task | D | `taskId`, `agentId`, `mode` (`AUTO`/`MANUAL`), `score?`, `breakdown?`, `previousAgentId?` |
| ASSIGNMENT_BLOCKED | task | D | `taskId`, `reason` (`NO_ELIGIBLE_AGENT`,...), `rejectedBy[]` (hard filter) |
| TASK_REASSIGNED | task | D | `taskId`, `fromAgentId`, `toAgentId`, `reason` (`reassign_failed`/`escalation`/`manual`) |
| TASK_HANDOVER | task | D | `taskId`, `fromAgentId`, `toAgentId`, `attempts`, `briefRef`, `branchRef?` (FR-111) |
| TASK_STARTED | task | D | `taskId`, `agentId`, `sessionId`, `attempt`, `branch?` |
| TASK_PROGRESS | task | D | `taskId`, `agentId`, `summary` (≤ 200 ký tự, đã redact), `percent?`, `stage?` (giới hạn tần suất, ≤ 1/30 s) |
| TASK_STATUS_CHANGED | task | D | `taskId`, `from`, `to` (10 `task_status`), `reason?` |
| TASK_BLOCKED | task | D | `taskId`, `blockedReason` (`DEPENDENCY_FAILED`,`NO_ELIGIBLE_AGENT`,`PROVIDER_DOWN`,`OWNERSHIP_CONFLICT`,`BUDGET`,`OPERATOR`,`EMERGENCY_STOP`) |
| TASK_REVIEW_REQUESTED | task | D | `taskId`, `reviewTaskId`, `reviewerRole`, `diffRef` |
| TASK_EVIDENCE_RECORDED | task | D | `taskId`, `evidenceId`, `criterionIndex`, `kind`, `artifactId?`, `refSha`, `verifiedBy{kind,id}`, `verdict` |
| TASK_COMPLETED | task | D | `taskId`, `agentId`, `evidenceIds[]`, `resultSummary`, `artifactIds[]`, `durationMs`, `attempts` — **chỉ phát khi evidence đủ** |
| TASK_FAILED | task | D | `taskId`, `agentId`, `attempt`, `failureClass`, `retryScheduled` (bool), `delayMs?`, `error?` (đã redact) |
| TASK_CANCELLED | task | D | `taskId`, `reason`, `cancelledBy` |
| TASK_REVISION_CREATED | task | D | `taskId`, `revision`, `fromApprovalId`, `briefRef` |

### 5.3 WORKFLOW_*

| Type | Aggregate | D/T | Payload |
|---|---|---|---|
| WORKFLOW_STARTED | run | D | `runId`, `workflowVersionId`, `projectId`, `triggeredBy`, `variablesDigest` |
| WORKFLOW_PAUSED / WORKFLOW_RESUMED | run | D | `runId`, `hard?` (bool), `by` |
| WORKFLOW_CANCELLED | run | D | `runId`, `reason`, `by` |
| WORKFLOW_COMPLETED | run | D | `runId`, `durationMs`, `revision`, `outputRef?` |
| WORKFLOW_FAILED | run | D | `runId`, `reason` (`REVISION_LIMIT`,`LOOP_LIMIT`,`NODE_FAILED`,...), `nodeRunId?` |
| WORKFLOW_STAGE_STARTED | run | D | `runId`, `stage`, `revision` |
| WORKFLOW_STAGE_COMPLETED | run | D | `runId`, `stage`, `revision`, `durationMs` |
| WORKFLOW_NODE_READY / _QUEUED / _STARTED | run | D | `runId`, `nodeRunId`, `nodeKey`, `kind`, `attempt`, `revision` |
| WORKFLOW_NODE_SUCCEEDED | run | D | như trên + `outputDigest`, `handle?` |
| WORKFLOW_NODE_FAILED | run | D | như trên + `failureClass`, `error?` |
| WORKFLOW_NODE_SKIPPED | run | D | `runId`, `nodeKey`, `reason` (`inactive_edges`) |
| WORKFLOW_NODE_RETRY_SCHEDULED | run | D | `nodeRunId`, `attempt`, `delayMs`, `failureClass` |
| WORKFLOW_LOOP_LIMIT_REACHED | run | D | `runId`, `edgeKey`, `maxIterations` |
| WORKFLOW_REVISION_STARTED | run | D | `runId`, `revision`, `gateNodeKey`, `fixTasks[{targetNodeKey,assignee}]` |
| WORKFLOW_CHECKPOINT_SAVED | run | D | `runId`, `seq`, `reason` |

Ghi chú: FR-117 dùng `STAGE_STARTED/STAGE_COMPLETED`; A07 dùng `WORKFLOW_STAGE_CHANGED`. Chọn `WORKFLOW_STAGE_STARTED/COMPLETED` (hai sự kiện rõ nghĩa cho office và timeline); alias `STAGE_STARTED` giữ trong ánh xạ FR (M-E2).

### 5.4 HUMAN_*

| Type | Aggregate | D/T | Payload |
|---|---|---|---|
| HUMAN_APPROVAL_REQUESTED | run (hoặc task) | D | `approvalId`, `nodeRunId?`, `taskId?`, `type` (`workflow_gate`,`tool_call`,`deploy`,`spend`,`memory_write`,...), `title`, `approverRoles[]`, `artifacts[]` (tham chiếu), `risk?`, `expiresAt?`, `mandatory` |
| HUMAN_APPROVED | run | D | `approvalId`, `decidedBy`, `decision` (`approve`,`approve_once`,`approve_always`), `comment?` |
| HUMAN_REJECTED | run | D | `approvalId`, `decidedBy`, `comment?`, `rejectItems[{file?,area?,severity,text,markupRef?}]` |
| HUMAN_APPROVAL_EXPIRED | run | D | `approvalId`, `onTimeout` (`reject`,`fail`,`escalate`,`auto_approve`) |
| HUMAN_FEEDBACK_SUBMITTED | run | D | `runId`, `deliverableRef`, `text`, `markupRefs[]` (FR-127, FR-130) |
| HUMAN_COMMAND | agent/task/run | D | `command` (`pause`,`resume`,`stop`,`retry`,`reassign`,`chat`), `targetRef`, `by` (FR-239; audit) |
| EMERGENCY_STOP | system | D | `by`, `reason`, `sessionsKilled`, `tasksBlocked` (FR-214) |

### 5.5 Event khác (FR-117 và hệ thống)

| Type | Aggregate | D/T | Payload |
|---|---|---|---|
| MEETING_STARTED / MEETING_ENDED | project | D | `meetingId`, `topic`, `participants[agentId]`, `round?` / `minutesRef`, `dissent?` |
| BUG_CREATED | task | D | `bugId`, `taskId?`, `severity`, `title`, `reportedBy` |
| BUILD_STARTED / BUILD_COMPLETED | run | D | `nodeRunId`, `targetEnv`, `passed?`, `reportRef?` |
| TEST_RESULT | run | D | `nodeRunId`, `passed`, `summary`, `reportRef` |
| PREVIEW_READY | run | D | `runId`, `previewUrl`, `commitSha` |
| WORKTREE_CREATED / WORKTREE_MERGED / WORKTREE_CONFLICT | task | D | `taskId`, `path`, `branch`, `baseRef`, `files[]?` (conflict) |
| RESOURCE_SEMAPHORE_WAIT | task | T | `taskId`, `layer` (`global`/`provider`/`project`/`agent`), `used`, `limit` |
| RESOURCE_CONCURRENCY_REDUCED | system | D | `from`, `to`, `reason` (`RATE_LIMIT`,`HOST_GUARD`) |
| PROVIDER_CONNECTED / PROVIDER_DISCONNECTED | provider | D | `providerId`, `kind`, `status`, `reason?` |
| SYSTEM_RECOVERED | system | D | `adopted`, `killed`, `reallocated`, `approvalsKept`, `durationMs` |

### 5.6 Ánh xạ danh sách cũ

| Nguồn | Tên cũ | Tên chuẩn |
|---|---|---|
| FR-117 | `AGENT_STATE_CHANGED` | giữ nguyên |
| A07 | `AGENT_STATUS_CHANGED` (5 giá trị) | `AGENT_STATE_CHANGED` (12 giá trị) |
| FR-117 | `MESSAGE_SENT` | `AGENT_MESSAGE_SENT` |
| FR-117 | `STAGE_STARTED/COMPLETED` | `WORKFLOW_STAGE_STARTED/COMPLETED` |
| A07 | `NODE_*` | `WORKFLOW_NODE_*` |
| A07 | `REVISION_CREATED` | `WORKFLOW_REVISION_STARTED` (+ `TASK_REVISION_CREATED` cho từng fix task) |
| A07 | `SEMAPHORE_WAIT` | `RESOURCE_SEMAPHORE_WAIT` |
| DB §5.2 | `session.started` … `office.animation` (dotted, lowercase) | Bảng dưới |

DB §5.2 dùng dạng dotted (`task.status_changed`, `run.node_entered`, `approval.requested`...). Hai kiểu đặt tên cùng tồn tại trong tài liệu nguồn; chọn **UPPER_SNAKE** vì FR-117 là yêu cầu của owner. Quy đổi: `session.*→AGENT_SESSION_*`, `task.*→TASK_*`, `run.*→WORKFLOW_*`, `approval.*→HUMAN_APPROVAL_*`, `agent.*→AGENT_*`, `llm.*`/`tool.*`/`file.*`/`shell.*`/`git.*→PROVIDER_*` (xem §6), `office.moved/animation` là **transient, không thuộc catalogue durable** (§9) (M-E2).

## 6. Event provider chuẩn hóa

Mục tiêu: một bộ event `PROVIDER_*` bất kể Claude Code, Codex hay OpenHands, để UI, office, usage không phụ thuộc tên tool/provider. Adapter chịu trách nhiệm chuẩn hóa; **webview không hardcode tên tool** (Pixel Agents: `providerCapabilities` mang `readingTools`, `subagentToolNames` xuống UI).

### 6.1 Danh mục

| Type | D/T | Payload | Ghi chú |
|---|---|---|---|
| PROVIDER_SESSION_STARTED | D | `sessionId`, `providerKind`, `providerSessionId?`, `cwd`, `model?`, `source` (`spawn`/`resume`/`adopt`) | |
| PROVIDER_SESSION_ENDED | D | `sessionId`, `reason` (`stop`,`exit`,`error`,`timeout`,`killed`), `exitCode?` | |
| PROVIDER_TURN_STARTED / PROVIDER_TURN_ENDED | D (sampled) | `sessionId`, `turnId`, `awaitingInput?`, `durationMs?` | `awaitingInput=true` ≈ Waiting for input |
| PROVIDER_TOOL_STARTED | D | `sessionId`, `toolCallId`, `toolName`, `toolClass` (`read`,`write`,`exec`,`web`,`task`,`other`), `inputDigest`, `securityRisk?` (`LOW/MEDIUM/HIGH/UNKNOWN`), `llmResponseId?` | `inputDigest` đã redact, không phải input thô |
| PROVIDER_TOOL_COMPLETED | D | `sessionId`, `toolCallId`, `actionId?`, `ok`, `durationMs`, `exitCode?`, `outputRef?` (artifact nếu > 8 KB) | Ghép cặp qua `toolCallId` |
| PROVIDER_TOOL_DENIED | D | `sessionId`, `toolCallId`, `by` (`policy`,`hook`,`user`), `reason` | Ví dụ ghi ngoài ownership |
| PROVIDER_PERMISSION_REQUESTED / _RESOLVED | D | `sessionId`, `requestId`, `toolName`, `risk`, `approvalId?` / `decision`, `by` | Liên kết `HUMAN_APPROVAL_REQUESTED` |
| PROVIDER_MESSAGE | D (sampled) | `sessionId`, `role` (`assistant`,`user`,`system`), `summary` (≤ 300 ký tự đã redact), `activatedSkills[]?` | Không lưu toàn văn |
| PROVIDER_OUTPUT_DELTA | T | `sessionId`, `itemId`, `order`, `kind` (`text`,`reasoning`), `content` | Chỉ Redis; không replay (OpenHands `delta`) |
| PROVIDER_USAGE | D | `sessionId`, `turnId?`, `tokens{in,out,cacheRead,cacheWrite,reasoning?}`, `contextTokens?`, `costUsdList?`, `usageId` | Ghi vào `usage_records`/ledger; `costBasis` luôn `list` (không phải tiền thật) |
| PROVIDER_RATE_LIMITED | D | `providerId`, `sessionId?`, `until?`, `source` (`parsed`,`default_backoff`), `consecutive` | Kích hoạt đóng băng pool |
| PROVIDER_ERROR | D | `sessionId?`, `providerId`, `class` (`AUTH`,`CRASH`,`TIMEOUT`,`UNKNOWN`), `message` (đã redact) | |
| PROVIDER_SUBAGENT_STARTED / _ENDED | D | `sessionId`, `subagentId`, `parentToolCallId?`, `label?` | |

### 6.2 Bảng chuẩn hóa từ từng provider

| PROVIDER_* | Claude Code (hook/SDK; Pixel Agents §2.1) | Codex (`exec --json`/app-server) | OpenHands (agent-server) |
|---|---|---|---|
| SESSION_STARTED | `SessionStart`, SDK `system/init` | `thread.started` | `POST /api/conversations` → state `idle/running` |
| SESSION_ENDED | `SessionEnd` | process exit / `turn.failed` | `ExecutionStatus` finished/error |
| TURN_STARTED | (không có hook riêng; suy ra từ tool/assistant đầu turn) | `turn.started` [Giả định tên] | `ExecutionStatus=running` |
| TURN_ENDED | `Stop` (`awaitingInput` khi `Notification(idle_prompt)`) | `turn.completed` | `ExecutionStatus=idle/finished` |
| TOOL_STARTED | `PreToolUse`, hoặc `tool_use` trong transcript | `item.started` (kind tool/command) | `ActionEvent` (`kind`, `tool_call_id`) |
| TOOL_COMPLETED | `PostToolUse`/`PostToolUseFailure` | `item.completed` | `ObservationEvent` (`action_id`) |
| TOOL_DENIED | `canUseTool` từ chối, hook `blocked` | sandbox/approval từ chối | `UserRejectObservation`, `HookExecutionEvent{blocked}` |
| PERMISSION_REQUESTED | `PermissionRequest`, `Notification(permission_prompt)` | approval request của app-server | `waiting_for_confirmation` |
| MESSAGE | `assistant` text trong transcript | `item.completed` (agent message) | `MessageEvent` |
| OUTPUT_DELTA | stream-json partial | stream item delta | `delta` frame |
| USAGE | `message.usage`, `result.usage` | `turn.completed` usage | `ConversationStateUpdateEvent{key:'stats'}` |
| RATE_LIMITED | `is_error` + "usage limit", `api_error_status` 429/529 | lỗi 429/usage limit [Giả định] | `ServerErrorEvent` |
| SUBAGENT_* | `SubagentStart/Stop`, `Task` tool | — | `TaskAction` / child conversation |

Ghi chú: `UserPromptSubmit` và `TaskCreated` của Claude **không** được thu thập (Pixel Agents cố ý bỏ để không lấy prompt text). Cột Codex là [Giả định] vì Codex CLI chưa cài và mọi kiểm thử Codex thực chưa chạy; tên event `thread.started`, `turn.completed`, `item.*` là [Đã xác minh] từ tài liệu Codex (`SKILL_SYSTEM.md` §3.3), tên `turn.started`/`turn.failed` chưa.

### 6.3 Từ PROVIDER_* sang AGENT_/TASK_*

Adapter chỉ phát `PROVIDER_*`. Các service miền **dịch** thành event miền: ví dụ `PROVIDER_SESSION_STARTED` đầu tiên → `AgentStateService` chuyển `assigned → running` → `AGENT_STATE_CHANGED` + `TASK_STARTED`. Office và UI đọc event miền cho trạng thái cấp cao, đọc `PROVIDER_TOOL_*` cho hoạt động chi tiết (typing vs reading).

## 7. Vận chuyển: outbox, Redis pub/sub, Redis Stream, thứ tự, replay

### 7.1 Outbox giao dịch [Đề xuất] — [Cần người dùng phê duyệt]

Luồng: `BEGIN` → thay đổi trạng thái nghiệp vụ → `INSERT agent_events` (cấp `sequence`, `global_seq`) → `INSERT event_outbox(event_id, global_seq)` → `COMMIT`. Relay đọc outbox, publish, xóa dòng.

- **Vì sao bảng outbox riêng**: `agent_events` append-only, role API chỉ có `INSERT, SELECT` (DB §5.1) nên không thể đánh dấu `published` như A07; cần bảng outbox mà relay được `DELETE`.
- Relay: **một leader** (advisory lock) đọc `ORDER BY global_seq LIMIT 500`, giữ thứ tự, phát **at-least-once**; leader chết thì replica khác lấy lock. A07 dùng nhiều relay với `FOR UPDATE SKIP LOCKED` (nhanh hơn nhưng mất thứ tự); chọn thứ tự vì office/timeline cần (M-E3).
- DB §5.4 đề xuất (a) outbox cho server mode, (b) publish-after-commit cho desktop. Đề xuất hợp nhất: **dùng outbox ở mọi chế độ** (D-014); chi phí nhỏ, loại một nhánh mã. Owner duyệt.
- Transient không đi qua outbox: publish thẳng Redis.

### 7.2 Redis pub/sub (đẩy tức thời)

Kênh theo DB §10: `aiw:ch:org:{orgId}:events` (mọi durable sau commit), `aiw:ch:session:{id}`, `aiw:ch:task:{id}`, `aiw:ch:approvals:{orgId}`, `aiw:ch:office:{layoutId}` (transient), `aiw:ch:control:agent:{id}` (lệnh server → runner, không phải event). Socket.IO dùng adapter Redis với kênh riêng `aiw:sio:*`. Room: `org:<id>`, `project:<id>`, `run:<id>`, `agent:<id>`, `task:<id>`. Pub/sub **không** đảm bảo giao; bù bằng replay (§7.5).

### 7.3 Redis Stream (đệm và consumer group)

- `aiw:evt:session:{sessionId}:buffer`, `MAXLEN ~ 2000` (DB §10): đệm reconnect < 10 phút cho console.
- `aiw:evt:org:{orgId}:stream` (**đề xuất thêm**; A07 gọi `events:<workspaceId>`): mọi durable, `MAXLEN ~` cấu hình (mặc định 50 000), dành cho consumer group: `office-projector`, `analytics`, `webhook`. Consumer ack theo `id`; xử lý **idempotent theo `event.id`** (at-least-once).
- Stream **không** là nguồn sự thật: mất Redis thì rebuild từ `agent_events` (xem `MULTI_AGENT_ORCHESTRATION.md` §10).
- Redis cấu hình `maxmemory-policy noeviction` (DB §10).

### 7.4 Thứ tự

| Phạm vi | Bảo đảm |
|---|---|
| Trong một aggregate | `sequence` liên tục, tăng dần (khóa dòng khi cấp); consumer phát hiện gap khi `sequence > last+1` và gọi REST bù |
| Giữa các aggregate | Theo `global_seq` **gần đúng**: bigserial không đảm bảo thứ tự commit tuyệt đối (DB §5.1); consumer chấp nhận out-of-order trong cửa sổ 1-2 s. Relay một leader giảm nhưng không loại bỏ |
| Quan hệ nhân quả | `causationId` + `correlationId` (không dựa vào thời gian) |
| Thời gian | `occurredAt` do server gán; đồng hồ đồng nhất theo `now()` Postgres (A07 §10) |

### 7.5 Replay bằng `lastEventId` [Đề xuất]

Hai loại con trỏ, tương tự `after_seq` của OpenHands (đã thay cặp `resend_mode/after_timestamp` vì so sánh timestamp lỗi):

| Feed | Con trỏ | Nguồn |
|---|---|---|
| Org/dashboard/office | `lastEventId = globalSeq` (số) | `agent_events WHERE global_seq > ?` (index `global_seq`) |
| Console một session | `(sessionId, afterSeq)` | `agent_events` theo `(session_id, sequence)` |

Giao thức (Socket.IO; SSE dùng `Last-Event-ID`):
1. Client kết nối `auth{token, lastEventId?}` và chọn room.
2. Server gửi frame `sync{fromSeq, throughSeq}` rồi phát lại durable `global_seq > lastEventId` (lọc theo room và quyền), **trang 500**, mỗi frame `durable{seq, event}`; transient không phát lại (`transient{event}` chỉ live).
3. Reconnect < 10 phút: đọc Redis Stream trước; fallback DB khi stream đã trim.
4. `lastEventId` cũ hơn retention hoặc stream đã trim: server trả `replay_truncated{oldestAvailable}`; client **tải snapshot** REST (trạng thái hiện tại của agent/task/run/office) rồi nối live từ `globalSeq` hiện tại.
5. Client bỏ event có `id` hoặc `sequence` đã thấy; phát hiện gap theo aggregate thì gọi lại REST `afterSeq`.
6. Rebuild projection (office, analytics): đọc `XRANGE`, thiếu thì `agent_events` theo `global_seq`.
7. Frame chưa biết (`type` lạ) bị bỏ qua, không lỗi.

## 8. Redaction

Thực hiện **trước khi persist** (một `EventRedactor` duy nhất trong pipeline: provider event → chuẩn hóa → redact → zod validate → ghi), và **lần nữa khi đọc** theo quyền người xem.

| Loại dữ liệu | Quy tắc | Cách xử lý |
|---|---|---|
| Credential (token OAuth, API key, `.credentials.json`/`auth.json`, `Authorization`) | D-008: app không đọc/lưu | Không bao giờ vào event; pattern quét (API key, JWT, PEM, `AKIA...`) → thay `«REDACTED:kind»`, ghi đường dẫn vào `redactions[]` |
| Biến môi trường, secret khai báo trong `secrets` | Danh sách key nhạy cảm của org | Loại khỏi `inputDigest`, `lastLine`, `error` |
| Prompt người dùng thô | Không thu thập (như Pixel Agents bỏ `UserPromptSubmit`) | Chỉ lưu `digest` (sha256) + độ dài; nội dung nằm ở task/message có kiểm soát quyền |
| stdout/stderr, nội dung file | Không nhét vào payload | > 8 KB hoặc bất kỳ log dài → artifact `log`, payload giữ `{artifactId, offset, length}` (DB §5.1) |
| Đường dẫn | Chuẩn hóa về tương đối workspace; che thư mục home | `C:\Users\<name>\...` → `~\...` |
| Thông tin cá nhân (email, tên) | Chỉ ở trường được khai báo | Che trong `summary`/`error` bằng regex; không đảm bảo hoàn toàn [Giả định] |
| Nội dung `memory`/`rejectItems` | Chứa text người dùng | Giữ nguyên nhưng gắn nhãn `sensitivity`; ẩn với vai `viewer` |

Ràng buộc: redact **không đảo ngược**; sự kiện kiểm toán bảo mật (FR-213: lệnh, file, network, permission, approval, emergency stop) vẫn ghi đầy đủ metadata nhưng không ghi giá trị secret. `redactions[]` giúp phát hiện khi pattern mới cần thêm.

## 9. Ánh xạ event → Virtual Office

### 9.1 Quy tắc bất biến [Đã xác minh yêu cầu FR-160/FR-118; Đề xuất cách cưỡng chế]

> Animation và di chuyển của nhân vật **chỉ** được tạo ra từ event thật đã persist (hoặc event transient do chính adapter phát từ hoạt động thật). Không có animation ngẫu nhiên, mô phỏng hay "trang trí chuyển động" ngoài event.

Cưỡng chế:
1. **Một writer duy nhất**: `OfficeProjector` (consumer group `office-projector`) là thành phần duy nhất sinh lệnh office (`aiw:ch:office:{layoutId}`: `office.moved`, `office.animation`).
2. Mỗi lệnh có `causedByEventId` (id của event nguồn) và `causedByType`; lệnh không có nguồn bị từ chối ở validator.
3. **Cấm wander ngẫu nhiên**: FSM nhân vật của Pixel Agents (§4.4: `IDLE --(wanderTimer)--> WALK(random)`, bubble/pet FOLLOW ngẫu nhiên) là **tham khảo**, phải **bỏ** phần ngẫu nhiên khi port (D-011). `Idle` là tư thế tĩnh, chỉ lặp khung hình sprite khi state `idle` **đang đúng** theo event.
4. Không event → văn phòng **đứng yên** (chấp nhận về UX; FR rủi ro đã ghi). Không có chế độ demo trong bản phát hành; fixture dev-only không được đóng gói [Cần người dùng phê duyệt].
5. Mất event (gap/replay): không phát lại lịch sử animation; **đồng bộ snapshot**: đặt nhân vật vào tư thế suy ra từ trạng thái hiện tại (`agents.status`, task, vị trí lưu ở `character_positions`), mọi chuyển động tiếp theo do event mới.
6. "Stale/mất tín hiệu" chỉ hiển thị khi server đã phát event tương ứng (Reaper → `AGENT_STATE_CHANGED`/`WORKFLOW_NODE_FAILED STALE`), không do timer phía UI.
7. Test: (a) property test mọi lệnh office có `causedByEventId` tồn tại; (b) luồng event rỗng → không lệnh nào; (c) cùng chuỗi event (replay) cho cùng trạng thái cuối.

### 9.2 Bảng ánh xạ mặc định [Đề xuất; cấu hình được theo FR-162]

13 animation (FR-158): Idle, Walk, Run, Sit, Work, Typing, Reading, Thinking, Meeting, Talking, Celebrating, Error, Waiting. Phòng (FR-151): Reception, Management, BA, Architecture, Design Studio, FE, BE, DB, QA, Security, DevOps, Meeting, Relax, Coffee. "Bàn" = bàn của agent trong phòng phòng ban (FR-157).

| Event (điều kiện) | Vị trí / hành động | Animation | Bubble |
|---|---|---|---|
| AGENT_CREATED | Hiệu ứng spawn tại Reception rồi (nếu có) đi tới phòng phòng ban | Walk → Idle | Tên + vai trò |
| AGENT_STATE_CHANGED → `idle` | Đi tới Relax hoặc Coffee (FR-164; **chỉ** khi có event này) | Walk → Idle | — |
| → `available` | Về bàn | Walk → Sit | — |
| → `assigned` | Về bàn nếu chưa ở đó | Walk → Sit | "Nhận: <task>" |
| → `running` | Tại bàn | **Theo hoạt động** (hàng dưới); mặc định Work | Tiêu đề task |
| → `reviewing` | Tại bàn | Reading | "Review: <task>" |
| → `waiting` | Tại bàn | Waiting | Lý do (`approval`, `quota`, `message`) |
| → `blocked` / `failed` | Tại bàn | Error | Lý do ngắn |
| → `completed` | Tại bàn | Celebrating (hiệu ứng ngắn) | "Xong: <task>" |
| → `paused` | Đứng tại chỗ | Waiting | Biểu tượng pause |
| → `disabled` | Mờ/ẩn | — | — |
| TASK_STARTED | Walk tới bàn → làm việc (ví dụ FR-162) | Walk → Typing | — |
| TASK_ASSIGNED (priority `critical`) | Đi nhanh tới bàn | Run → Sit | "Khẩn" |
| PROVIDER_TURN_STARTED, chưa có tool đang chạy | Tại bàn | Thinking | — |
| PROVIDER_OUTPUT_DELTA `kind=reasoning` | Tại bàn | Thinking | — |
| PROVIDER_TOOL_STARTED `toolClass=write`/`exec` | Tại bàn | Typing | Tên tool rút gọn |
| PROVIDER_TOOL_STARTED `toolClass=read`/`web` | Tại bàn | Reading | Tên tool rút gọn |
| PROVIDER_TOOL_COMPLETED / PROVIDER_TURN_ENDED | Về tư thế theo `AGENT_STATE_CHANGED` hiện hành | (giữ 300 ms để khỏi nhấp nháy, như `TOOL_DONE_DELAY_MS`) | — |
| PROVIDER_PERMISSION_REQUESTED | Tại bàn | Waiting | "..." vàng |
| PROVIDER_RATE_LIMITED | Tại bàn | Waiting | Đếm ngược tới `until` |
| PROVIDER_ERROR | Tại bàn | Error | Loại lỗi |
| PROVIDER_SUBAGENT_STARTED | Nhân vật phụ cạnh cha | Walk → Work | Nhãn subagent |
| AGENT_MESSAGE_SENT | Cả hai tại chỗ; không di chuyển | Talking (người gửi), bubble | Loại message |
| TASK_HANDOVER | Người gửi đi tới bàn người nhận | Walk → Talking | "Bàn giao" |
| TASK_COMPLETED | Tại bàn | Celebrating | "Xong" |
| TASK_FAILED | Tại bàn | Error | `failureClass` |
| MEETING_STARTED | Participants tới phòng Meeting | Walk → Meeting | Chủ đề |
| MEETING_ENDED | Về bàn | Walk → Sit | "Minutes" |
| HUMAN_APPROVAL_REQUESTED | Agent chờ tại bàn; badge ở Reception/Management (UI) | Waiting | "Chờ duyệt" |
| HUMAN_APPROVED | Theo `AGENT_STATE_CHANGED` kế tiếp | — | "Đã duyệt" |
| HUMAN_REJECTED | Agent liên quan | Thinking | "Revision R<n>" |
| WORKFLOW_STAGE_STARTED | Nhấn mạnh phòng tương ứng (trang trí UI) | (không animation nhân vật) | Biển stage |
| TEST_RESULT `passed=false`, BUG_CREATED | Agent QA tại bàn | Error | "Bug: <title>" |
| PREVIEW_READY | Agent DevOps tại bàn | Celebrating | "Preview" |
| EMERGENCY_STOP | Mọi agent tại chỗ | Error/Waiting | "STOP" |

Nhận xét đối chiếu: Pixel Agents §2.3 ghi "Thinking không có tín hiệu riêng"; ở ta, Thinking dựa trên `PROVIDER_TURN_STARTED` + không tool, hoặc delta `reasoning`, tức vẫn là event thật. OI-FR-06 (meeting thật hay animation): chỉ phát `MEETING_STARTED` khi bản ghi họp bất đồng bộ thật được mở (`AGENT_COLLABORATION.md` §7), nên animation Meeting luôn có nguồn thật. Bảng này lưu ở dữ liệu cấu hình (không hardcode), cho phép người dùng sửa.

## 10. Retention

| Dữ liệu | Thời gian giữ | Cơ chế | Nguồn |
|---|---|---|---|
| `agent_events` (partition theo tháng) | 90 ngày desktop / 365 ngày server (`events.retention_days`) | `DETACH PARTITION CONCURRENTLY` rồi `DROP`; tùy chọn export `.jsonl.gz` trước (`events.archive_on_drop`) | DB §5.3 |
| `event_outbox` | Chỉ tới khi publish (xóa sau publish) | Relay; dòng tồn đọng > 1 phút cảnh báo | Đề xuất |
| Redis stream org | `MAXLEN ~ 50 000` (cấu hình) | trim gần đúng | Đề xuất |
| Redis stream session | `MAXLEN ~ 2000` | trim | DB §10 |
| A07: replay từ outbox 7 ngày | **Thay bằng** replay từ `agent_events` theo retention ở trên | — | M-E3 |
| Event bảo mật/audit (FR-213) | Không theo retention events; sao sang `audit_logs` | Ghi kép trong transaction | Đề xuất |
| `usage_records` | Không drop; roll-up; archive > 24 tháng | DB §5.3 | DB |
| Transient (delta, office) | Không lưu | — | — |
| `task_context_snapshot`/evidence | Suốt vòng đời project | — | `AGENT_COLLABORATION.md` §10 |

`AGENT_HEARTBEAT` transient nên không tăng khối lượng bảng; `TASK_PROGRESS` và `PROVIDER_MESSAGE` được giới hạn tần suất để event store không phình.

## 11. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| ES-01 | Tên event UPPER_SNAKE theo tiền tố; envelope tách payload | FR-117 (owner), OpenHands | [Đề xuất] |
| ES-02 | Một outbox giao dịch ở mọi chế độ; relay một leader | D-014; giữ thứ tự; `agent_events` append-only | [Đề xuất] — [Cần người dùng phê duyệt] |
| ES-03 | Pub/sub để đẩy; replay để bù; Stream làm đệm/consumer group, không phải nguồn sự thật | Pub/sub không bền | [Đề xuất] |
| ES-04 | `lastEventId = globalSeq` cho feed tổng; `(sessionId, afterSeq)` cho console | Tránh so sánh timestamp lỗi (OpenHands) | [Đề xuất] |
| ES-05 | Event provider chuẩn hóa `PROVIDER_*`; service miền dịch thành event miền | UI/office độc lập provider | [Đề xuất] |
| ES-06 | Redact trước khi persist; không lưu prompt/stdout thô | D-008, FR-212/213 | [Đề xuất] |
| ES-07 | Office chỉ từ event thật; mọi lệnh office có `causedByEventId`; bỏ wander | FR-160, D-011 | [Đã xác minh] yêu cầu; [Đề xuất] cách cưỡng chế |
| ES-08 | Heartbeat là transient; liveness bền ở cột `last_heartbeat_at` | Tránh phình bảng | [Đề xuất] |
| ES-09 | Audit bảo mật ghi kép sang `audit_logs` | Không bị retention events xóa | [Đề xuất] |

## 12. Dependencies

`DATABASE_DESIGN.md` (`agent_events`, `agent_sessions`, `audit_logs`, `character_positions`; thêm `event_outbox`, `event_sequences`, cột `delivery`/`redactions` hoặc quy ước trong payload), Redis 7 (Streams, noeviction), Socket.IO + `@socket.io/redis-adapter`, `packages/contracts` (zod), `AGENT_ARCHITECTURE.md`, `WORKFLOW_ENGINE.md`, adapter docs (`CLAUDE_CODE_INTEGRATION.md`, `CODEX_INTEGRATION.md`, `OPENHANDS_ANALYSIS.md`), `PIXEL_AGENTS_ANALYSIS.md`/D-011 (Phaser), `SECURITY.md` (redaction, audit).

## 13. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Khối lượng event lớn (tool call, usage) làm phình `agent_events` | Giới hạn tần suất `TASK_PROGRESS`/`PROVIDER_MESSAGE`; stdout vào artifact; partition + retention |
| Out-of-order giữa aggregate làm office/timeline sai | Cửa sổ chịu lệch 1-2 s; `causationId`; consumer idempotent |
| Redaction sót secret | Pattern + danh sách key org + `redactions[]`; test với fixture rò rỉ; không ghi input thô |
| Outbox leader chết làm trễ phát | Advisory lock failover, cảnh báo backlog, replay bù |
| Hai chuẩn tên (UPPER vs dotted) gây rối khi triển khai | Bảng quy đổi §5.6; zod schema là nguồn duy nhất |
| Office "tĩnh" khi ít event gây khó hiểu | Chấp nhận (FR rủi ro); badge trạng thái từ snapshot; không giả lập |
| Thiếu `delivery`/`redactions` ở schema DB | Chốt cách lưu trước migration |
| Tên event Codex chưa kiểm | Adapter Codex sau khi cài CLI; hợp đồng test bằng fixture ghi lại |

## 14. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Outbox ở mọi chế độ (§7.1) so với đề xuất tách hai chế độ của DB §5.4.
2. [Cần người dùng phê duyệt] Danh mục event FR-117 (AP-FR-03) và tên chuẩn UPPER_SNAKE.
3. [Cần người dùng phê duyệt] Có cho phép chế độ mô phỏng/demo cho office không (FR-160 nói không giả lập).
4. [Cần người dùng phê duyệt] `event_sequences` cho aggregate ngoài session (DB §5.1 đang chờ duyệt).
5. UUID v7 (cần hàm app hoặc PG 18) so với `gen_random_uuid()`; hay giữ v4 và dùng `global_seq` để sắp xếp.
6. Có lưu `PROVIDER_TURN_*`/`PROVIDER_MESSAGE` ở mức sampled hay chỉ transient.
7. Quyền xem event theo vai (viewer không thấy `rejectItems`, `summary`?).
8. Webhook/consumer ngoài (analytics, webhook) ngoài MVP; cần xác định bảo mật nếu có.

### Mâu thuẫn nguồn liên quan

| ID | Mâu thuẫn | Xử lý |
|---|---|---|
| M-E1 | `id`: A07 ULID; DB `uuid` | UUID v7 (hoặc v4 + `global_seq`) |
| M-E2 | Tên event: FR-117, A07 (`NODE_*`, `AGENT_STATUS_CHANGED`), DB dotted lowercase | §5.6 bảng quy đổi, UPPER_SNAKE |
| M-E3 | Outbox: A07 `EventOutbox.published` + SKIP LOCKED (nhiều relay), replay 7 ngày; DB §5.4 hai chế độ (a)/(b), retention 90/365 ngày | §7.1, §10 |
| M-E4 | Stream: A07 `events:<workspaceId>`; DB `aiw:evt:session:{id}:buffer` | §7.3 giữ cả hai, đặt tên theo tiền tố `aiw:` |
| M-E5 | `AGENT_OUTPUT_CHUNK` (A07) vs `llm.stream_chunk` (DB, chỉ debug) | `PROVIDER_OUTPUT_DELTA` transient |
| M-E6 | `session.heartbeat` durable (DB §5.2) vs A07 heartbeat throttle 5 s chỉ WS | Transient; liveness ở cột |
| M-E7 | Pixel Agents có wander ngẫu nhiên; FR-160 cấm | Bỏ khi port (§9.1) |
| M-E8 | `actor`: DB chỉ `actor_user_id`; A07 `actor{kind,id}` | Envelope giữ `actor`; DB lưu thêm trong payload hoặc cột mới |

## 15. Acceptance Criteria (Phase 1)

- [ ] Zod schema cho mọi `type` ở §5-§6; test hợp lệ/không hợp lệ; consumer bỏ qua `type` lạ mà không lỗi.
- [ ] Mỗi transition state ghi đúng một event cùng transaction; chèn lỗi giữa chừng không để lại event mồ côi hay state không có event.
- [ ] Outbox: kill relay giữa batch → không mất event, có thể trùng nhưng consumer idempotent theo `event.id`; thứ tự `global_seq` giữ nguyên khi chỉ một leader.
- [ ] Replay: client ngắt 5 phút nhận lại đủ durable theo `lastEventId` không trùng; ngắt quá retention nhận `replay_truncated` và tự đồng bộ snapshot; gap `sequence` kích hoạt REST bù.
- [ ] Redis flush: Stream/pub/sub rỗng, hệ thống rebuild từ `agent_events`, UI vẫn hiển thị đúng sau reconnect.
- [ ] Redaction: bộ fixture chứa API key, JWT, đường dẫn home, env → không xuất hiện trong `agent_events`, Redis, hay log; `redactions[]` ghi đúng đường dẫn; stdout > 8 KB nằm trong artifact.
- [ ] Chuẩn hóa provider: fixture Claude (hook + transcript) cho ra chuỗi `PROVIDER_*` đúng; cặp `PROVIDER_TOOL_STARTED/COMPLETED` ghép đúng `toolCallId`; fixture Codex/OpenHands bổ sung khi có.
- [ ] Office: property test mọi lệnh có `causedByEventId` hợp lệ; luồng event rỗng không sinh lệnh nào; replay cùng chuỗi event cho cùng trạng thái cuối; không còn đường mã wander ngẫu nhiên.
- [ ] Bảng ánh xạ §9.2 nằm trong cấu hình dữ liệu, đổi được mà không sửa mã; mọi animation trong FR-158 đều có ít nhất một event kích hoạt hoặc được đánh dấu "chưa dùng".
- [ ] Retention: partition cũ bị drop theo `events.retention_days`; `audit_logs` giữ nguyên; `usage_records` không bị drop.
- [ ] Owner đã duyệt các mục [Cần người dùng phê duyệt] ở §14.
