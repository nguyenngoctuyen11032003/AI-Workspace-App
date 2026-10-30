# AGENT_SCHEDULER.md — Dependency-aware Agent Scheduler

Tác giả: Agent 13 — Phase 0, chỉ tài liệu. Ngày: 2026-10-30. Trạng thái: DRAFT.
Liên quan: `AGENT_ORGANIZATION.md` (role/skill/ownership), `RESOURCE_MANAGEMENT.md` (limits, budgets, semaphores), `AGENT_COLLABORATION.md` (message bus), `docs/research/MULTI_AGENT_SCALABILITY.md` (số đo).
Workflow engine (DAG definition, BullMQ vs LangGraph, state machine của workflow): **xem `ORCHESTRATION_WORKFLOW_RESEARCH.md` (A07)**. Tài liệu này chỉ giải quyết câu hỏi: *khi workflow engine nói "task T sẵn sàng", agent nào chạy, khi nào, với tài nguyên gì, và chuyện gì xảy ra khi hỏng.*

Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].

---

## 1. Mục đích

Thiết kế **Agent Scheduler** — thành phần nằm giữa workflow engine (A07) và provider adapters (A05/A06): nhận task ở trạng thái READY, chọn agent phù hợp theo chuyên môn, xin slot tài nguyên, spawn execution, theo dõi heartbeat, xử lý pause/resume/cancel/retry, tái phân bổ khi agent hỏng, giữ công bằng giữa nhiều project dưới limit chung, và phục hồi sau crash.

## 2. Phạm vi

- Trong: ready-set, priority, scoring, task ownership, queue/backpressure, vòng đời execution, retry/backoff, reallocation, fairness, heartbeat/stale, crash recovery, pseudocode TypeScript.
- Ngoài: định nghĩa DAG và trạng thái workflow (A07); giá trị limit cụ thể và cost accounting (`RESOURCE_MANAGEMENT.md`); cách spawn/parse output CLI (A05/A06); schema SQL (A08).

## 3. Vị trí trong kiến trúc [Đề xuất]

```
Workflow Engine (A07) --(TaskReady event)--> Agent Scheduler --(acquire)--> Resource Manager (Redis semaphores)
                                                   |--(spawn)--> Provider Adapter (claude-code | codex) --> CLI subprocess
                                                   |--(persist)--> Postgres: task, agent_execution, scheduler_event
                                                   |--(emit)--> Event Bus (WS gateway, message bus, metrics)
```

Nguyên tắc: **Postgres là nguồn sự thật**, Redis chỉ giữ trạng thái tạm (semaphore, heartbeat TTL, lease). Scheduler chạy như một NestJS module với một "tick loop" (interval 1–2 s) + phản ứng theo event; cả hai đường đều idempotent.

## 4. Mô hình dữ liệu logic [Đề xuất] (schema cuối do A08)

```ts
type TaskState = 'DRAFT'|'BLOCKED'|'READY'|'QUEUED'|'ASSIGNED'|'RUNNING'|'PAUSED'
               | 'WAITING_APPROVAL'|'REVIEW'|'DONE'|'FAILED'|'CANCELLED';

interface Task {
  id: string; projectId: string; workflowRunId?: string;
  title: string; description: string;
  requiredRole?: string;            // template code, optional
  requiredSkills: string[]; preferredSkills: string[];
  complexity: 1|2|3|4|5;            // ước lượng (PM agent hoặc user)
  priority: number;                 // 0..100, cao = quan trọng
  deadlineAt?: Date;
  dependsOn: string[];              // task ids (engine A07 cũng giữ; scheduler chỉ đọc)
  writePatterns: string[]; readPatterns: string[];
  estimatedMinutes?: number; estimatedTokens?: number;
  budgetUsd?: number;
  ownerAgentId?: string;            // DUY NHẤT một agent chịu trách nhiệm
  state: TaskState; attempt: number; maxRetries: number;
  providerConstraint?: 'claude-code'|'codex'|null;
}

interface AgentExecution {
  id: string; taskId: string; agentId: string; projectId: string;
  provider: string; model?: string; credentialId: string;
  pid?: number; providerSessionId?: string;     // session_id từ CLI JSON
  state: 'SPAWNING'|'RUNNING'|'PAUSED'|'STOPPING'|'DONE'|'FAILED'|'KILLED'|'LOST';
  leaseToken: string; leaseExpiresAt: Date;      // Redis lease mirror
  startedAt: Date; lastHeartbeatAt: Date; endedAt?: Date;
  tokensIn: number; tokensOut: number; costUsd: number;
  exitCode?: number; failureClass?: FailureClass;
}

type FailureClass = 'RATE_LIMIT'|'AUTH'|'TIMEOUT'|'CRASH'|'TOOL_ERROR'|'BUDGET_EXCEEDED'|'USER_CANCEL'|'STALE'|'UNKNOWN';
```

## 5. Thuật toán lập lịch [Đề xuất]

### 5.1 Ready-set computation

Task vào ready-set khi: `state ∈ {READY, QUEUED}` AND tất cả `dependsOn` ở `DONE` (engine A07 đã đảm bảo khi phát `TaskReady`, nhưng scheduler **kiểm lại** trong cùng transaction để chống race) AND không bị `PAUSED` ở cấp project/org AND budget project chưa cạn.

```ts
async function computeReadySet(db: Db): Promise<Task[]> {
  return db.query(`
    SELECT t.* FROM task t
    WHERE t.state IN ('READY','QUEUED')
      AND NOT EXISTS (SELECT 1 FROM task d JOIN task_dep td ON td.dep_id = d.id
                      WHERE td.task_id = t.id AND d.state <> 'DONE')
      AND NOT EXISTS (SELECT 1 FROM project p WHERE p.id = t.project_id AND p.paused)
    ORDER BY t.priority DESC, t.deadline_at NULLS LAST, t.created_at ASC
    FOR UPDATE SKIP LOCKED`);   // nhiều scheduler worker không giẫm nhau
}
```

### 5.2 Priority hiệu dụng

`effectivePriority = priority + urgency(deadline) + criticalPathBoost + ageBoost − projectFairnessPenalty`

- `urgency`: +20 nếu deadline < 2h, +10 nếu < 24h.
- `criticalPathBoost`: +15 nếu task có ≥ 3 task phụ thuộc trực tiếp vào nó (unblock nhiều việc).
- `ageBoost`: +1 mỗi 10 phút chờ, tối đa +20 (chống starvation).
- `projectFairnessPenalty`: xem §9.

### 5.3 Assignment scoring (specialization-based)

Với mỗi task trong ready-set, ứng viên = instance `ACTIVE` không bị DISABLED, có `templateVersion` hợp lệ. Mỗi tiêu chí trả về 0..1 hoặc **hard reject**.

| Tiêu chí | Loại | Cách tính | Trọng số mặc định |
|---|---|---|---|
| Role match | hard nếu `requiredRole` có; soft nếu không | 1 nếu template.code == requiredRole; 0.5 nếu cùng department; 0 khác | 0.25 |
| Skills | hard cho `requiredSkills` | Σ weight(skill ∩ preferred) / Σ weight(preferred) | 0.20 |
| Task complexity vs model tier | soft | complexity 4–5 cần tier `premium`/`standard`; 1–2 ưu tiên `economy` (rẻ) | 0.10 |
| Priority alignment | soft | task priority cao → ưu tiên agent có `successRate` cao | 0.05 |
| Dependencies continuity | soft | +1 nếu agent này đã làm task cha (context đã có) | 0.10 |
| Provider availability | **hard** | credential pool không ở trạng thái `RATE_LIMITED`/`DOWN`; `providerConstraint` khớp | — |
| Workload | soft | 1 − runningExecutions/maxConcurrentExecutions (instance limit, thường 1) ; hard reject nếu = 0 | 0.10 |
| Workspace availability | **hard** | `task.writePatterns` không giao với writePatterns của execution đang RUNNING khác; glob của task ⊆ ownershipPatterns của template | — |
| Permissions | **hard** | permissionLevel(instance) ≥ level task yêu cầu | — |
| Estimated time | soft | ưu tiên agent có `avgMinutesPerComplexity` gần `estimatedMinutes` (tránh over/under-kill) | 0.05 |
| Resource budget | **hard** | budget còn lại của agent/project ≥ `estimatedCost(task)` | — |
| Recent failure | soft | −0.3 nếu agent vừa fail task này (attempt trước) → ưu tiên agent khác | 0.15 (penalty) |

```ts
function scoreCandidate(task: Task, agent: AgentInstance, ctx: SchedCtx): number | null {
  if (task.requiredRole && agent.template.code !== task.requiredRole) return null;
  if (!task.requiredSkills.every(s => agent.skillSet.has(s))) return null;
  if (!ctx.providerPool(agent.credentialId).isAvailable()) return null;
  if (!permits(agent.permissionLevel, task)) return null;
  if (ctx.workspaceConflicts(task.writePatterns, agent.id)) return null;
  if (!coversOwnership(agent.template.ownershipPatterns, task.writePatterns)) return null;
  if (ctx.budget.remaining(agent, task.projectId) < ctx.estimateCost(task, agent)) return null;
  if (agent.running >= agent.limits.maxConcurrentExecutions) return null;

  const w = ctx.weights; // configurable per org
  let s = 0;
  s += w.role   * (task.requiredRole ? 1 : roleAffinity(task, agent));
  s += w.skills * preferredSkillCoverage(task, agent);
  s += w.tier   * tierFit(task.complexity, agent.modelTier);
  s += w.prio   * (task.priority / 100) * agent.stats.successRate;
  s += w.cont   * (agent.id === ctx.parentOwner(task) ? 1 : 0);
  s += w.load   * (1 - agent.running / agent.limits.maxConcurrentExecutions);
  s += w.time   * timeFit(task.estimatedMinutes, agent.stats.avgMinutes[task.complexity]);
  s -= w.fail   * (ctx.failedBefore(task, agent) ? 1 : 0);
  return s;
}
```

Tie-break: ít execution gần nhất (LRU) → chi phí ước tính thấp hơn → id ổn định (deterministic cho test).

### 5.4 Task ownership — một agent chịu trách nhiệm

- Khi assign: `task.ownerAgentId = agent.id` trong cùng transaction với tạo `AgentExecution`. Review/help từ agent khác **không** đổi owner (xem `AGENT_COLLABORATION.md` §4).
- Reassign chỉ xảy ra qua: (a) failure reallocation (§8), (b) operator/PM đề xuất + orchestrator thực hiện. Mọi lần đổi owner ghi `scheduler_event(kind='REASSIGN', from, to, reason)`.
- Một agent có thể sở hữu nhiều task nhưng chỉ **executing** `maxConcurrentExecutions` (mặc định 1 — một CLI session/agent, để log và context không trộn).

## 6. Queue và backpressure [Đề xuất]

- Hàng đợi logic = ready-set đã sắp xếp; không cần queue riêng vì Postgres `SKIP LOCKED` đủ cho ≤ vài nghìn task. Nếu A07 chọn BullMQ thì scheduler là consumer của queue `task.ready` với concurrency = 1 worker/tick (quyết định cuối: A07).
- Backpressure theo thứ tự kiểm: `global.max_concurrent_agents` → `provider[credential].max_concurrent` → `project.max_concurrent` → `agent.maxConcurrentExecutions`. Hết slot → task giữ `QUEUED`, ghi `queuedReason`, hiển thị trên dashboard ("chờ slot global (5/5)").
- Admission control: khi `queueDepth > 3 × max_concurrent_agents` trong > 10 phút → cảnh báo operator; không tự tăng limit.
- Không spawn khi RAM free < `min_free_ram_mb` (default 2048) hoặc CPU load trung bình 1 phút > 90% (số đo thực nghiệm: mỗi process ~230 MB RSS, spawn 3 cùng lúc đẩy CPU lên 79% trên i5-11400H — [Đã xác minh], xem `MULTI_AGENT_SCALABILITY.md`).

## 7. Vòng đời execution và thao tác điều khiển [Đề xuất]

```
QUEUED --assign--> ASSIGNED --spawn ok--> RUNNING --exit 0 + output ok--> REVIEW/DONE
                                  |--spawn fail--> FAILED(retry?)
RUNNING --pause--> PAUSED --resume--> RUNNING       (Claude: resume bằng `--resume <session_id>`; A05 xác nhận)
RUNNING --approval needed--> WAITING_APPROVAL --approve--> RUNNING
RUNNING --cancel--> STOPPING --(SIGTERM, 10s, then taskkill /T /F)--> CANCELLED
RUNNING --heartbeat stale--> LOST --> reallocation (§8)
```

- **Pause**: với CLI không có tín hiệu pause an toàn trên Windows; cách làm: gửi cancel mềm (đóng stdin / SIGTERM), lưu `providerSessionId`, đặt `PAUSED`; **Resume** = spawn lại với `claude --resume <session_id> -p "continue"` [Giả định — A05 cần xác minh flag resume với `-p`]. Với Codex: [Giả định] chờ A06.
- **Cancel**: SIGTERM → chờ `graceful_stop_timeout` (10 s) → `taskkill /PID <pid> /T /F` (kill cả cây, vì CLI spawn thêm child cho tool). Ghi `failureClass='USER_CANCEL'`, không retry.
- **Retry**: chỉ tự động cho `RATE_LIMIT`, `TIMEOUT`, `CRASH`, `STALE`, `TOOL_ERROR` (nếu idempotent). Không tự retry `AUTH`, `BUDGET_EXCEEDED`, `USER_CANCEL`.

```ts
function backoffMs(attempt: number, cls: FailureClass, base = 30_000, cap = 15 * 60_000): number {
  const mult = cls === 'RATE_LIMIT' ? 4 : 1;          // rate-limit: chờ lâu hơn
  const exp = Math.min(cap, base * mult * 2 ** (attempt - 1));
  return exp * (0.8 + Math.random() * 0.4);            // jitter ±20%
}
```

Với `RATE_LIMIT` kèm `retry-after`/reset time (nếu CLI trả về — [Giả định] cần kiểm thử thật), dùng thời điểm đó thay vì backoff, và **đóng băng cả credential pool** chứ không chỉ task.

## 8. Tự động tái phân bổ khi agent hỏng [Đề xuất]

Trigger: execution → `FAILED` (retryable) hoặc `LOST`.

1. Giải phóng semaphore/lease; đánh dấu workspace của task "dirty" (có thể có file ghi dở).
2. Chạy `workspaceRecovery`: nếu task dùng git worktree → `git status`; nếu có thay đổi → stash vào `reallocation/<taskId>/<attempt>` để agent mới quyết định dùng hay bỏ; ghi link vào task context.
3. `task.attempt++`; nếu `attempt > maxRetries` → `FAILED` vĩnh viễn, notify owner.
4. Ngược lại `state=READY`, `scheduledNotBefore = now + backoff`; scoring lần sau phạt agent vừa fail (−0.15×) → thường chọn agent khác cùng role nếu có; nếu chỉ có 1 agent của role đó, vẫn gán lại cho nó sau backoff.
5. Nếu `failureClass='AUTH'` hoặc provider `DOWN` → không reallocate tự động; task về `BLOCKED(reason=provider)`, chờ operator (đổi provider cần approval — `RESOURCE_MANAGEMENT.md` §9).

## 9. Công bằng nhiều project dưới limit chung [Đề xuất]

- Mỗi project có `weight` (default 1) và `max_concurrent` (default = global). Slot global phân theo **weighted fair share**: `fairShare(p) = global × weight(p) / Σ weight(active projects)`.
- `projectFairnessPenalty(p) = 25 × max(0, running(p) − fairShare(p))` — project đang vượt phần của mình bị trừ điểm ưu tiên, nhưng **không bị chặn cứng** nếu project khác không có task READY (work-conserving).
- Starvation guard: task chờ > 30 phút với `ageBoost` max vẫn không được slot → log cảnh báo `FAIRNESS_STARVATION`.
- Operator có thể "pin" một project ở chế độ `exclusive` (chiếm toàn bộ slot) — hành động có audit.

## 10. Heartbeat và phát hiện stale [Đề xuất]

- Nguồn heartbeat (bất kỳ nguồn nào cập nhật `lastHeartbeatAt`): (1) stream-json event từ stdout của CLI (A05: `--output-format stream-json` phát event mỗi tool call/message), (2) file thay đổi trong workspace (watcher), (3) process còn sống (`pid` tồn tại) — nguồn (3) **chỉ chứng minh sống, không chứng minh tiến triển**.
- Ngưỡng: `stale_soft` = 5 phút không có (1)/(2) → đánh dấu "no progress", hiển thị vàng; `stale_hard` = `task_timeout` hoặc 20 phút không có (1)/(2) khi process vẫn sống → `LOST`, kill cây process, reallocate.
- Lease: mỗi execution giữ Redis key `lease:exec:<id>` TTL 60 s, scheduler worker gia hạn mỗi 20 s khi process còn sống. Lease hết hạn mà DB còn `RUNNING` → coi là scheduler worker đã chết (không phải agent chết) → bước recovery §11.

```ts
async function tick(ctx: SchedCtx) {
  await detectStale(ctx);                 // §10
  await ctx.db.tx(async (tx) => {
    const ready = await computeReadySet(tx);
    for (const task of ready) {
      if (task.scheduledNotBefore && task.scheduledNotBefore > ctx.now()) continue;
      const cands = await ctx.registry.activeInstances(task.projectId);
      const scored = cands.map(a => [a, scoreCandidate(task, a, ctx)] as const)
                          .filter(([, s]) => s !== null)
                          .sort(([a, s1], [b, s2]) => (s2! - s1!) || ctx.tieBreak(a, b));
      if (!scored.length) { await markQueued(tx, task, 'NO_ELIGIBLE_AGENT'); continue; }
      const [agent] = scored[0];
      const slot = await ctx.resources.tryAcquire({ projectId: task.projectId, credentialId: agent.credentialId, agentId: agent.id });
      if (!slot) { await markQueued(tx, task, slot?.reason ?? 'NO_SLOT'); continue; }
      const exec = await createExecution(tx, task, agent, slot.leaseToken);     // task.state=ASSIGNED, ownerAgentId=agent.id
      ctx.afterCommit(() => ctx.spawner.spawn(exec));                          // spawn ngoài transaction
    }
  });
}
```

## 11. Crash recovery [Đề xuất]

Trạng thái bền trong Postgres: `task`, `agent_execution`, `scheduler_event`, `resource_ledger`. Redis chỉ có: semaphore counters, lease TTL, heartbeat cache. **Redis có thể xóa trắng bất cứ lúc nào.**

Boot sequence của scheduler:
1. Đọc mọi `agent_execution` ở `SPAWNING|RUNNING|PAUSED|STOPPING`.
2. Với mỗi execution có `pid`: kiểm process còn sống (`Get-Process -Id`/`tasklist`) **và** command line chứa `--session-id`/workspace của execution (tránh PID reuse) [Giả định: cần A05 xác nhận CLI cho phép truyền `--session-id` để nhận dạng]. Nếu sống → **adopt**: gắn lại stdout nếu được (không thể nếu pipe đã mất — khi đó chỉ theo dõi file log + pid); nếu không gắn được, chính sách mặc định: kill + reallocate (an toàn hơn adopt mù).
3. Nếu chết → `state=FAILED(failureClass='CRASH')` → reallocation §8.
4. **Rebuild semaphores**: `SET sem:global = COUNT(RUNNING)`, `sem:provider:<cred> = COUNT(RUNNING by cred)`, `sem:project:<p>`… Chỉ sau khi rebuild xong mới bật tick loop.
5. Phát `SchedulerRecovered` kèm thống kê (adopted, killed, reallocated) để dashboard hiển thị.

Idempotency: mọi spawn ghi `execution.id` làm `--session-id` (nếu CLI hỗ trợ) hoặc ghi vào env `AIW_EXECUTION_ID` của subprocess để tránh spawn đôi sau crash.

## 12. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Scheduler tách khỏi workflow engine | Engine (A07) có thể là BullMQ hoặc LangGraph; scheduler không phụ thuộc lựa chọn đó | [Đề xuất] |
| Postgres `FOR UPDATE SKIP LOCKED` cho ready-set | Đơn giản, không cần queue riêng cho quy mô ≤ 18 agent | [Đề xuất] |
| Scoring có hard-reject + soft weights cấu hình per org | Cho phép tổ chức tùy biến mà không đổi code | [Đề xuất] |
| 1 execution/agent mặc định | Log/context sạch; đo được 1 process ≈ 230 MB RSS nên không thiếu RAM, lý do là chất lượng không phải tài nguyên | [Đề xuất], số đo [Đã xác minh] |
| Rate-limit đóng băng cả credential pool | Credential dùng chung cho mọi agent (xác minh `subscriptionType: team`, 1 OAuth) | [Đã xác minh] + [Đề xuất] |
| Kill + reallocate thay vì adopt mù sau crash | Không thể gắn lại stdout pipe trên Windows; adopt mù mất evidence | [Đề xuất] |

## 13. Dependencies

A07 (TaskReady event contract, state machine), A05 (stream-json events, `--resume`, `--session-id`, exit codes), A06 (Codex tương đương), A08 (schema), A09 (permission check hook), `RESOURCE_MANAGEMENT.md` (semaphore API), A11 (dashboard fields).

## 14. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Race giữa engine phát TaskReady và scheduler tick | Kiểm lại dependencies trong transaction; event chỉ là "gợi ý tick sớm" |
| PID reuse trên Windows sau crash | Kiểm command line/env trước khi adopt; mặc định kill + reallocate |
| Pause/resume không hỗ trợ bởi CLI non-interactive | Fallback: cancel + resume session; nếu `--resume` không hoạt động với `-p`, pause = cancel + retry từ đầu (mất tiến độ) — cần A05 kiểm thử |
| Scoring quá phức tạp → khó debug "vì sao chọn agent X" | Ghi `scheduler_event(kind='SCORE', breakdown JSON)` cho mọi quyết định; dashboard hiển thị |
| Stale detection sai với task dài hợp lệ (build lớn) | Task khai `expectedSilenceMinutes`; file watcher là nguồn heartbeat thứ hai |

## 15. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Khi không có agent nào đủ skill, có cho phép scheduler "hạ chuẩn" (bỏ qua preferredSkills, chọn cùng department) tự động hay phải hỏi? Đề xuất: tự động hạ chuẩn chỉ với `complexity ≤ 2`.
2. Có cho phép >1 execution/agent (ví dụ Docs agent chạy 3 task nhỏ song song)? Đề xuất: cho phép qua `limits.maxConcurrentExecutions` nhưng mặc định 1.
3. Cơ chế pause thật sự của CLI trên Windows — chờ A05/A06.
4. A07 chọn BullMQ hay LangGraph quyết định scheduler là consumer hay tick-loop thuần.

## 16. Acceptance Criteria

- [ ] Unit test scoring: với fixture 18 template, task `requiredRole=backend` luôn chọn backend instance; task không có requiredRole và `requiredSkills=[playwright]` chọn test_automation.
- [ ] Test hard-reject: workspace conflict (2 task cùng `apps/api/**`) không chạy song song.
- [ ] Test backpressure: `max_concurrent_agents=2`, 5 task READY → đúng 2 RUNNING, 3 QUEUED với `queuedReason`.
- [ ] Test fairness: 2 project weight 1:1, 10 task mỗi bên, limit 4 → mỗi project ≥1 RUNNING trong suốt run.
- [ ] Test stale: giả lập process sống nhưng không output 20 phút (clock mock) → LOST → reallocate với attempt+1, agent khác được ưu tiên.
- [ ] Test crash recovery: xóa Redis giữa chừng → semaphore rebuild đúng COUNT(RUNNING); không spawn đôi.
- [ ] Mọi quyết định assign có `scheduler_event` với breakdown điểm.
- [ ] Cancel giết toàn bộ cây process (kiểm bằng `tasklist` sau 15 s không còn child).
