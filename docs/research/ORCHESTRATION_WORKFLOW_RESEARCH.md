# ORCHESTRATION & WORKFLOW ENGINE — Research & Design Recommendation

**Agent 07 — Multi-Agent Orchestration & Workflow Engine Architect**
**Ngày:** 2026-10-30 · **Stack mục tiêu:** NestJS + TypeScript + PostgreSQL (Prisma) + Redis + BullMQ · **Platform:** Windows 11

---

## 0. Tóm tắt điều hành (Executive Summary)

**Khuyến nghị: Custom DAG Executor trên BullMQ + Postgres-persisted run state** (gọi tắt là **"Conductor"**), với:

- **Postgres** là *source of truth* duy nhất cho `workflow_runs`, `node_runs`, `approvals`, `checkpoints`, `events`.
- **BullMQ** chỉ đóng vai trò *transport* + *retry/backoff* + *stalled-job detection*; **không** dùng BullMQ Flows làm cấu trúc DAG (Flows là *tree*, không biểu diễn được diamond/merge, và không có "wait hours for human").
- **Scheduler** (NestJS service, idempotent, chạy được nhiều replica) làm *dependency resolver* kiểu ready-set; semaphore global 10 + per-provider trong Redis (Lua script).
- **Approval gate** = node_run ở trạng thái `WAITING_HUMAN`, không giữ bất kỳ job/worker nào; resume = ghi quyết định vào Postgres rồi kích scheduler.
- **LangGraph JS không thắng rõ ràng**: `interrupt()` re-run node từ đầu, Send API + interrupt cùng lúc có hành vi không rõ, graph phải compile từ code (không tự nhiên cho React Flow JSON user-editable), và không có concurrency cap xuyên process. Dùng LangGraph **bên trong** một Agent node (nếu muốn) chứ không làm engine.
- **Temporal** là lựa chọn kỹ thuật mạnh nhất nhưng kéo theo Temporal Server (4 service + DB + optional Elasticsearch) — chi phí vận hành không tương xứng với 10 agent đồng thời.

Bài học rút ra từ **ChatDev 2.0**: layered-topological execution, edge `trigger` flags, cycle = super-node với `max_iterations`, edge `condition`/`process` — nhưng ChatDev chạy in-memory bằng thread, human input là `Future` timeout 1800s, **không có durable checkpoint**; đây chính là chỗ chúng ta phải làm khác.
Bài học từ **OpenHands**: event stream là *append-only log* có `kind` discriminator (ActionEvent / ObservationEvent / ConversationStateUpdateEvent / PauseEvent / UserRejectObservation…), delegation = *child conversation* với task brief tự chứa (child không thấy lịch sử parent) → đây là mẫu cho Agent Message Bus token-economy của chúng ta.

---

## 1. Comparison Matrix

Yêu cầu (R1–R10):
R1 Durable state trong Postgres · R2 Resume sau process restart · R3 Human approval chờ giờ/ngày · R4 Parallel fan-out, cap 10 global + per-provider · R5 Retry exponential backoff · R6 Cancel/Pause/Resume · R7 Branching conditions · R8 Graph user-editable từ React Flow JSON · R9 Versioned workflow definitions · R10 Event emission cho UI.

| Req | LangGraph JS (StateGraph + PostgresSaver) | **BullMQ + Custom DAG + Postgres (Conductor)** | Temporal TS | XState v5 + BullMQ |
|---|---|---|---|---|
| R1 Postgres durable | ✅ PostgresSaver (checkpoint blob per super-step, schema riêng của LangGraph, khó query/report) | ✅ Schema do ta kiểm soát, query được bằng SQL thường | ⚠️ Postgres là persistence của *Temporal Server*, không phải của app; app phải tự mirror sang DB của mình | ⚠️ `getPersistedSnapshot()` → JSON blob; tự lưu |
| R2 Resume sau restart | ✅ invoke lại cùng `thread_id`; node bị cắt dở **chạy lại từ đầu node** | ✅ Scheduler đọc `node_runs` → tính ready-set; job BullMQ stalled → requeue | ✅ Replay deterministic — mạnh nhất | ⚠️ Restore snapshot OK, nhưng side effect đang chạy dở phải tự xử lý |
| R3 Approval chờ ngày | ✅ `interrupt()` + `Command({resume})`; cảnh báo: code trước `interrupt()` chạy lại khi resume → phải idempotent | ✅ Node ở `WAITING_HUMAN`, không giữ job; resume bằng 1 UPDATE + enqueue. Không giới hạn thời gian | ✅ `condition()` + signal, timer tới hàng tháng | ✅ State `waitingApproval`; event `APPROVE` từ API |
| R4 Fan-out + cap 10 global + per-provider | ⚠️ `Send` fan-out trong 1 process; **không có** cross-process semaphore; cap phải tự làm ngoài | ✅ Redis Lua semaphore `sem:global`, `sem:provider:claude`, `sem:provider:codex`; queue khi hết slot | ✅ Child workflows + worker `maxConcurrentActivityTaskExecutions`; per-provider = task queue riêng | ⚠️ Parallel states có, cap phải ở BullMQ |
| R5 Retry backoff | ⚠️ `retryPolicy` per node (in-process) | ✅ BullMQ `attempts` + `backoff: {type:'exponential'}` + custom `backoffStrategy` | ✅ RetryPolicy activity | ✅ qua BullMQ |
| R6 Cancel/Pause/Resume | ⚠️ Không có cancel native; phải abort signal + bỏ checkpoint | ✅ Trạng thái run `PAUSED/CANCELLING`; worker check cancel flag + kill subprocess; `queue.pause()` cho global | ✅ `cancel`, `pause` qua signal | ✅ Event-driven |
| R7 Branching | ✅ `addConditionalEdges` | ✅ Condition node + edge `when` expression (JSONLogic / jexl sandboxed) | ✅ code | ✅ guards |
| R8 Graph từ React Flow JSON | ❌ Graph phải build bằng code; có thể build động từ JSON nhưng mất type-safety và phải compile lại mỗi run; subgraph/Send khó map 1:1 | ✅ JSON **là** định nghĩa; executor interpret trực tiếp | ❌ Workflow là code TS deterministic; interpreter-over-JSON làm được nhưng mất lợi thế | ⚠️ Machine config JSON được, nhưng DAG ↔ statechart không tự nhiên |
| R9 Versioned definitions | ⚠️ Tự làm | ✅ `workflow_definitions(version)`, run pin `definition_version_id` | ⚠️ `patched()` API phức tạp | ⚠️ Tự làm |
| R10 Event cho UI | ⚠️ stream events, phải bridge sang WS | ✅ Outbox table → Redis pub/sub → Socket.IO | ⚠️ Poll history / interceptor | ✅ `actor.subscribe` |
| **Ops cost** | Thấp (thư viện) | **Thấp** (đã có Redis+PG) | **Cao**: Temporal Server (frontend/history/matching/worker) + DB + (optional) Elasticsearch; hoặc Temporal Cloud trả phí | Thấp |
| **Fit với 10 AI employees + git worktrees** | Trung bình | **Cao** | Cao nhưng overkill | Trung bình |

**Inngest / Trigger.dev**: Trigger.dev v3 self-host (Apache-2.0, Docker Compose: Postgres + Redis + worker + dashboard) với `wait.forToken()`/`task.triggerAndWait()`; Inngest có `step.waitForEvent()` nhưng self-host production ràng buộc license/enterprise (nguồn mâu thuẫn, cần verify). Cả hai đều là *durable function runner* — ta lại cần *graph interpreter user-editable*; chúng chỉ thay được lớp BullMQ, không thay được DAG executor. Kết luận: **không dùng**, giữ BullMQ (đã có trong stack, ít moving parts).

**XState v5**: giữ lại **ở phạm vi nhỏ** — làm state machine cho *lifecycle của 1 node_run* và *1 workflow_run* (typed transitions, guard), không làm engine toàn cục.

---

## 2. Kiến trúc khuyến nghị — "Conductor"

```
┌──────────────┐   POST /runs        ┌────────────────────────────────┐
│ React Flow UI│ ───────────────────▶│ NestJS API (WorkflowModule)    │
│  (builder)   │ ◀── WS events ──────│  - DefinitionService (validate)│
└──────────────┘                     │  - RunService (start/pause/…)  │
                                     │  - ApprovalService             │
                                     └──────┬───────────────▲─────────┘
                                            │ enqueue        │ events
                     ┌──────────────────────▼────────────────┴──────────┐
                     │ Scheduler (BullMQ queue "scheduler", 1 job/run,   │
                     │  jobId = run_id → idempotent, debounce)           │
                     │  tick(run): load node_runs → ready-set →          │
                     │   acquire semaphores → enqueue node jobs          │
                     └───────┬──────────────────┬──────────────┬─────────┘
                             ▼                  ▼              ▼
                     queue "node.agent"   queue "node.tool"  queue "node.test"/"node.deploy"
                     (Worker concurrency   (Worker conc 20)  (sandboxed processor)
                      10, provider-aware)
                             │
                     ┌───────▼───────────────────────────────────────────┐
                     │ AgentRunner: spawn claude/codex CLI in worktree,  │
                     │ stream stdout → events, heartbeat → Redis,        │
                     │ on exit → UPDATE node_runs → enqueue scheduler    │
                     └───────────────────────────────────────────────────┘
        Postgres: workflow_definitions · workflow_runs · node_runs · node_attempts ·
                  approvals · checkpoints · agent_messages · events (outbox)
        Redis:    BullMQ queues · sem:* (Lua) · heartbeat:* · pubsub events
```

### 2.1 Nguyên tắc thiết kế

1. **Postgres là nguồn sự thật; Redis/BullMQ là tạm thời và tái tạo được.** Mất Redis → scheduler quét `node_runs WHERE status IN ('READY','RUNNING')` và re-enqueue. (ChatDev mất toàn bộ state khi process chết; ta tránh điều đó.)
2. **Mọi chuyển trạng thái là một transaction + outbox event** (cùng 1 `BEGIN`/`COMMIT`). Không có "emit event rồi mới UPDATE".
3. **Scheduler là hàm thuần idempotent**: `tick(runId)` có thể gọi 100 lần cho cùng kết quả. BullMQ job `jobId = sched:${runId}` tự dedupe.
4. **Node worker không biết graph**; nó chỉ biết `node_run_id`. Graph logic tập trung ở scheduler → dễ test.
5. **Human gates không chiếm worker**. Đây là lý do số 1 không dùng "job chờ trong processor" (BullMQ lock sẽ stall sau 30s).
6. **Subprocess (Claude Code / Codex CLI) là activity không idempotent** → mỗi attempt có `attempt_no`, worktree riêng `wt/<run>/<node>/<attempt>`; retry = attempt mới, không "resume giữa chừng" subprocess.

### 2.2 Vì sao không LangGraph làm engine

- `interrupt()` yêu cầu node idempotent trước điểm ngắt; Agent node của ta là subprocess 10–40 phút — resume sẽ *chạy lại từ đầu*.
- `Send` fan-out chỉ trong 1 process → không thể cap 10 agent *xuyên* replica/worker.
- Cấu trúc graph là code; React Flow JSON → phải sinh `StateGraph` động mỗi run; versioning và diff giữa 2 version của user trở nên gián tiếp.
- PostgresSaver lưu blob (msgpack) → báo cáo "node nào đang chạy, bao lâu, bao nhiêu token" phải parse blob.
- **Giữ LangGraph ở cấp độ bên trong 1 node** (ví dụ: node "PM planning" dùng LangGraph ReAct nhỏ) — tùy Agent 05/06 quyết.

### 2.3 Vì sao không BullMQ Flows làm DAG

Flows = *tree*: parent chờ *tất cả* children; không có diamond (A→B, A→C, B→D, C→D). Không có condition edge, không "chờ human" mà không giữ job, và `getChildrenValues()` chỉ là map đơn giản. Ta dùng Flows **chỉ** cho một trường hợp: Parallel node fan-out *cố định* (FE/BE/DB) có thể dùng `FlowProducer` để gom kết quả, nhưng để đồng nhất thì scheduler tự quản lý Merge node — đơn giản hơn.

---

## 3. Data Model

### 3.1 Workflow Definition JSON (tương thích React Flow)

Lưu nguyên `nodes[]`/`edges[]` theo React Flow (`id, type, position, data`), **engine chỉ đọc `data`**. Validation bằng zod.

```ts
// packages/workflow-schema/src/definition.ts
import { z } from 'zod';

export const NodeType = z.enum([
  'start','end','agent','approval','condition','parallel','merge','tool','test','deployment',
]);

const RetryPolicy = z.object({
  maxAttempts: z.number().int().min(1).max(10).default(3),
  backoff: z.enum(['exponential','fixed']).default('exponential'),
  baseDelayMs: z.number().int().default(30_000),
  maxDelayMs: z.number().int().default(15 * 60_000),
  retryOn: z.array(z.enum(['TIMEOUT','PROVIDER_ERROR','RATE_LIMIT','NONZERO_EXIT'])).default(['TIMEOUT','PROVIDER_ERROR','RATE_LIMIT']),
});

const AgentNodeData = z.object({
  kind: z.literal('agent'),
  label: z.string(),
  stage: z.string().optional(),            // nhãn stage cho WORKFLOW_STAGE_CHANGED
  role: z.string(),                        // 'backend_dev' | 'frontend_dev' | 'qa' | 'pm' ...
  assignee: z.union([z.literal('auto'), z.string().uuid()]).default('auto'),
  provider: z.enum(['claude','codex','any']).default('any'),
  promptTemplate: z.string(),              // handlebars; context từ inputs
  inputs: z.array(z.string()).default([]), // keys lấy từ upstream outputs: '<nodeId>.<key>'
  outputs: z.array(z.string()).default(['summary','files','decisions']),
  timeoutMs: z.number().int().default(45 * 60_000),
  retry: RetryPolicy.default({}),
  worktree: z.object({ base: z.string().default('main'), isolate: z.boolean().default(true) }).default({}),
  permissions: z.array(z.string()).default([]), // 'git:push','deploy:staging'
});

const ApprovalNodeData = z.object({
  kind: z.literal('approval'),
  label: z.string(),
  stage: z.string().optional(),
  approverRoles: z.array(z.string()).min(1),
  reviewArtifacts: z.array(z.string()).default([]),   // keys upstream
  timeoutMs: z.number().int().nullable().default(null), // null = chờ vô hạn
  onTimeout: z.enum(['fail','auto_approve','escalate']).default('fail'),
  onReject: z.object({ mode: z.enum(['revision_loop','fail']).default('revision_loop'),
                       maxRevisions: z.number().int().default(3) }).default({}),
});

const ConditionNodeData = z.object({
  kind: z.literal('condition'),
  label: z.string(),
  stage: z.string().optional(),
  // JSONLogic — không eval JS tùy ý
  rules: z.array(z.object({ handle: z.string(), when: z.record(z.any()) })),
  defaultHandle: z.string().default('else'),
});

const ParallelNodeData = z.object({ kind: z.literal('parallel'), label: z.string(), stage: z.string().optional(),
  maxConcurrency: z.number().int().default(3) });

const MergeNodeData = z.object({ kind: z.literal('merge'), label: z.string(), stage: z.string().optional(),
  strategy: z.enum(['all','any','quorum']).default('all'), quorum: z.number().int().optional(),
  onBranchFailure: z.enum(['fail','continue']).default('fail') });

const ToolNodeData = z.object({ kind: z.literal('tool'), label: z.string(), stage: z.string().optional(),
  tool: z.string(), args: z.record(z.any()).default({}), timeoutMs: z.number().int().default(10*60_000), retry: RetryPolicy.default({}) });
const TestNodeData = z.object({ kind: z.literal('test'), label: z.string(), stage: z.string().optional(),
  command: z.string(), cwd: z.string().default('.'), passIfExitZero: z.boolean().default(true),
  timeoutMs: z.number().int().default(20*60_000), retry: RetryPolicy.default({ maxAttempts: 1 }) });
const DeploymentNodeData = z.object({ kind: z.literal('deployment'), label: z.string(), stage: z.string().optional(),
  target: z.enum(['staging','production']), script: z.string(), requiresPermission: z.string().default('deploy:staging') });

export const NodeData = z.discriminatedUnion('kind', [
  z.object({ kind: z.literal('start'), label: z.string() }),
  z.object({ kind: z.literal('end'), label: z.string() }),
  AgentNodeData, ApprovalNodeData, ConditionNodeData, ParallelNodeData, MergeNodeData,
  ToolNodeData, TestNodeData, DeploymentNodeData,
]);

export const RFNode = z.object({
  id: z.string(), type: NodeType,
  position: z.object({ x: z.number(), y: z.number() }),
  data: NodeData,
});
export const RFEdge = z.object({
  id: z.string(), source: z.string(), target: z.string(),
  sourceHandle: z.string().nullable().default(null),   // 'approved' | 'rejected' | condition handle
  targetHandle: z.string().nullable().default(null),
  data: z.object({
    carry: z.array(z.string()).default(['*']),   // outputs nào được truyền xuống
    loopBack: z.boolean().default(false),          // edge quay lại (fix loop)
  }).default({}),
});
export const WorkflowDefinition = z.object({
  schemaVersion: z.literal(1),
  name: z.string(),
  nodes: z.array(RFNode).min(2),
  edges: z.array(RFEdge),
  variables: z.record(z.any()).default({}),
});
```

**Validation rules** (DefinitionService): đúng 1 `start`, ≥1 `end`; mọi node reachable từ start; graph là DAG **ngoại trừ** edges `loopBack=true`; `condition` phải có edge cho mọi `handle` + default; `approval` phải có edge `sourceHandle='approved'` (edge `rejected` optional — nếu thiếu thì revision loop do PM tạo); mỗi `parallel` phải có đúng 1 `merge` dominator.

### 3.2 Prisma schema

```prisma
enum RunStatus     { PENDING RUNNING PAUSED WAITING_HUMAN COMPLETED FAILED CANCELLING CANCELLED }
enum NodeRunStatus { PENDING READY QUEUED RUNNING WAITING_HUMAN SUCCEEDED FAILED SKIPPED CANCELLED }
enum ApprovalDecision { PENDING APPROVED REJECTED EXPIRED }

model WorkflowDefinition {
  id          String   @id @default(uuid())
  workspaceId String
  name        String
  isTemplate  Boolean  @default(false)
  versions    WorkflowDefinitionVersion[]
  @@index([workspaceId])
}
model WorkflowDefinitionVersion {
  id           String   @id @default(uuid())
  definitionId String
  version      Int
  graph        Json      // WorkflowDefinition JSON (React Flow compatible)
  graphHash    String    // sha256(canonical json) — detect no-op saves
  createdBy    String
  createdAt    DateTime @default(now())
  definition   WorkflowDefinition @relation(fields: [definitionId], references: [id])
  @@unique([definitionId, version])
}
model WorkflowRun {
  id                  String    @id @default(uuid())
  workspaceId         String
  projectId           String
  definitionVersionId String     // PIN version → edit template không ảnh hưởng run đang chạy
  status              RunStatus @default(PENDING)
  input               Json
  variables           Json      @default("{}")
  currentStage        String?   // stage đang chạy, cho UI
  revision            Int       @default(0)
  priority            Int       @default(5)   // 1 cao … 9 thấp
  startedAt           DateTime?
  finishedAt          DateTime?
  cancelReason        String?
  nodeRuns            NodeRun[]
  @@index([workspaceId, status])
}
model NodeRun {
  id           String        @id @default(uuid())
  runId        String
  nodeId       String        // id trong graph JSON (hoặc 'dyn:…' cho node động)
  nodeKind     String
  status       NodeRunStatus @default(PENDING)
  revision     Int           @default(0)   // tăng khi PM tạo fix task
  attempt      Int           @default(0)
  assigneeId   String?       // agent employee id
  provider     String?
  input        Json?         // resolved context (sau khi áp carry)
  output       Json?         // { summary, files[], decisions[], blockers[], handle?, metrics }
  error        Json?
  bullJobId    String?
  worktree     String?
  heartbeatAt  DateTime?
  startedAt    DateTime?
  finishedAt   DateTime?
  run          WorkflowRun   @relation(fields: [runId], references: [id])
  attempts     NodeAttempt[]
  @@unique([runId, nodeId, revision])
  @@index([status, heartbeatAt])
}
model NodeAttempt {
  id         String   @id @default(uuid())
  nodeRunId  String
  attempt    Int
  status     String
  exitCode   Int?
  stdoutRef  String?  // path/object key tới log
  tokensIn   Int?     @default(0)
  tokensOut  Int?     @default(0)
  costUsd    Decimal? @db.Decimal(10,4)
  startedAt  DateTime @default(now())
  finishedAt DateTime?
  nodeRun    NodeRun  @relation(fields: [nodeRunId], references: [id])
  @@unique([nodeRunId, attempt])
}
model Checkpoint {
  id        String   @id @default(uuid())
  runId     String
  seq       Int
  reason    String   // 'node_succeeded' | 'approval_requested' | 'manual' | 'before_revision'
  snapshot  Json     // { nodeRuns: {nodeId: {status, outputDigest}}, variables, revision }
  createdAt DateTime @default(now())
  @@unique([runId, seq])
}
model Approval {
  id          String           @id @default(uuid())
  runId       String
  nodeRunId   String           @unique
  requestedAt DateTime         @default(now())
  expiresAt   DateTime?
  decision    ApprovalDecision @default(PENDING)
  decidedBy   String?
  decidedAt   DateTime?
  comment     String?
  rejectItems Json?            // [{ file?, area?, severity, text }] → PM dùng tạo fix tasks
  artifacts   Json             // snapshot những gì reviewer thấy
  @@index([decision, expiresAt])
}
model AgentMessage {   // xem §5
  id         String   @id @default(uuid())
  runId      String
  fromAgent  String
  toAgent    String?  // null = tới PM
  type       String   // HANDOFF | HELP_REQUEST | HELP_RESPONSE | REVIEW_REQUEST | REVIEW_RESULT | BLOCKER | INFO
  threadId   String?
  payload    Json     // schema cố định (TaskBrief)
  tokenEst   Int
  readAt     DateTime?
  createdAt  DateTime @default(now())
  @@index([runId, toAgent, readAt])
}
model EventOutbox {
  id        BigInt   @id @default(autoincrement())
  runId     String?
  type      String
  payload   Json
  createdAt DateTime @default(now())
  published Boolean  @default(false)
  @@index([published, id])
}
```

**Checkpoint vs node_runs**: `node_runs` đã là trạng thái đầy đủ (tương đương channel values của LangGraph); `Checkpoint` là *snapshot nén* để (a) "rewind về trước revision N", (b) audit, (c) clone run. Resume sau crash **không** cần checkpoint — chỉ cần `node_runs`.

---

## 4. Scheduler Design

### 4.1 Vòng lặp `tick(runId)`

```ts
@Processor('scheduler')
export class SchedulerProcessor extends WorkerHost {
  async process(job: Job<{ runId: string }>) {
    const { runId } = job.data;
    await this.prisma.$transaction(async (tx) => {
      // 1. Khóa run (SELECT ... FOR UPDATE) → chỉ 1 tick/run tại một thời điểm, dù có N replica
      const [run] = await tx.$queryRaw<WorkflowRun[]>`SELECT * FROM "WorkflowRun" WHERE id=${runId} FOR UPDATE`;
      if (!['RUNNING','WAITING_HUMAN'].includes(run.status)) return;

      const graph = await this.defs.load(run.definitionVersionId);     // cached by version id
      const nodeRuns = await tx.nodeRun.findMany({ where: { runId } });
      const state = latestByNodeId(nodeRuns);                            // bản revision mới nhất mỗi nodeId

      // 2. Tính ready-set
      const ready = graph.nodes.filter(n =>
        state[n.id]?.status === 'PENDING' && predecessorsSatisfied(n, graph, state));

      for (const node of ready) {
        const input = resolveInput(node, graph, state);               // áp edge.carry + node.inputs
        await tx.nodeRun.update({ where: { id: state[node.id].id }, data: { status: 'READY', input } });
        await this.outbox.emit(tx, 'NODE_READY', { runId, nodeRunId: state[node.id].id, nodeId: node.id });
      }
      await this.propagateSkips(tx, graph, state);                      // nhánh inactive → SKIPPED
      await this.updateStage(tx, run, graph, state);                    // WORKFLOW_STAGE_CHANGED
      if (allTerminal(graph, state)) await this.finish(tx, run, state); // COMPLETED / FAILED
    }, { isolationLevel: 'ReadCommitted' });

    // 3. Dispatch ngoài transaction (acquire semaphore + enqueue). Idempotent nhờ jobId.
    await this.dispatcher.dispatchReady(runId);
  }
}
```

`predecessorsSatisfied(node)`:
- **Mặc định (AND)**: mọi incoming edge không-loopBack phải có source ở trạng thái terminal; edge là *active* nếu source `SUCCEEDED` và (`sourceHandle` null hoặc trùng `output.handle` của source). Source `SKIPPED` hoặc handle khác → edge inactive. Node ready khi **≥1 edge active và không edge nào còn pending**; nếu *tất cả* incoming inactive → node `SKIPPED` (lan truyền — tương đương "not triggered → skipped" của ChatDev `DAGExecutor`).
- **Merge node**: theo `strategy` (`all`/`any`/`quorum`), chỉ đếm các nhánh thuộc parallel group; `onBranchFailure='fail'` → merge FAILED khi 1 nhánh FAILED hết attempt.
- **loopBack edge**: không tính vào điều kiện ready. Khi source của loopBack SUCCEEDED, scheduler *reset* target (và downstream tới node đích) về `PENDING` với `revision+1`, giới hạn `maxIterations=3` (hằng engine, như `loop_counter` của ChatDev).

### 4.2 Dispatcher: semaphore Redis (Lua, atomic)

```lua
-- acquire.lua  KEYS[1]=sem:global KEYS[2]=sem:provider:<p>
-- ARGV[1]=capGlobal ARGV[2]=capProv ARGV[3]=nodeRunId ARGV[4]=leaseTtlSec
local g = redis.call('SCARD', KEYS[1]); local p = redis.call('SCARD', KEYS[2])
if g >= tonumber(ARGV[1]) or p >= tonumber(ARGV[2]) then return 0 end
redis.call('SADD', KEYS[1], ARGV[3]); redis.call('SADD', KEYS[2], ARGV[3])
redis.call('SET', 'sem:lease:'..ARGV[3], KEYS[2], 'EX', ARGV[4])  -- lease hết hạn → reaper SREM
return 1
```

```ts
async dispatchReady(runId: string) {
  const ready = await this.prisma.nodeRun.findMany({
    where: { runId, status: 'READY' }, include: { run: true },
    orderBy: [{ run: { priority: 'asc' } }, { createdAt: 'asc' }],   // priority rồi FIFO
  });
  for (const nr of ready) {
    if (nr.nodeKind === 'approval') { await this.approvals.open(nr); continue; }   // không tốn slot
    if (['condition','parallel','merge','start','end'].includes(nr.nodeKind)) {
      await this.inlineNodes.run(nr);                                              // chạy trong API process, < 50ms
      await this.schedulerQueue.add('tick', { runId }, { jobId: `sched:${runId}` }); continue;
    }
    const provider = await this.assignment.pickProvider(nr);          // §6
    const ok = await this.sem.acquire(nr.id, provider, { global: 10, provider: CAP[provider] });
    if (!ok) { await this.outbox.emitNoTx('SEMAPHORE_WAIT', { nodeRunId: nr.id, provider }); continue; } // ở lại READY
    await this.prisma.nodeRun.update({ where: { id: nr.id }, data: { status: 'QUEUED', provider } });
    await this.nodeQueue(nr.nodeKind, provider).add(nr.nodeKind, { nodeRunId: nr.id }, {
      jobId: `node:${nr.id}:${nr.attempt}`,
      priority: nr.run.priority,
      attempts: nr.input.retry.maxAttempts,
      backoff: { type: 'custom' },
      removeOnComplete: 1000, removeOnFail: 5000,
    });
  }
}
```

Khi một node release slot (`SREM` + `DEL lease`) → `schedulerQueue.add('tick')` cho **mọi run có node READY** (`SELECT DISTINCT "runId" FROM "NodeRun" WHERE status='READY' LIMIT 50`). Đây là cơ chế queueing khi hết limit: node chờ trong Postgres (READY), không chờ trong BullMQ, nên không bị stalled, không giữ lock.

Per-provider cap đề xuất: `claude: 6`, `codex: 6`, global `10` (cấu hình ở `workspace_settings`, hot-reload qua Redis key `cfg:caps`).

### 4.3 Retry với exponential backoff + rate-limit awareness

- BullMQ `attempts` xử lý TIMEOUT/PROVIDER_ERROR/NONZERO_EXIT; mỗi attempt ghi `NodeAttempt`, `NodeRun.attempt` = attempt hiện tại; worktree của attempt cũ giữ 24h để debug.
- **Rate limit từ provider (HTTP 429 / "usage limit" trong stdout CLI)**: worker gọi `await worker.rateLimit(retryAfterMs)` + `throw Worker.RateLimitError()` → job về `waiting`, **không tiêu attempt**. Limiter của BullMQ là **global per queue** (BullMQ ≥3 bỏ `groupKey`), nên mỗi provider có **queue riêng**: `node.agent.claude`, `node.agent.codex` với `limiter: { max: 30, duration: 60_000 }`.
- Custom backoff strategy trên Worker:

```ts
new Worker('node.agent.claude', processor, {
  connection, concurrency: 6,
  limiter: { max: 30, duration: 60_000 },
  lockDuration: 60_000, lockRenewTime: 20_000, stalledInterval: 30_000, maxStalledCount: 2,
  settings: {
    backoffStrategy: (attempts, _type, err) => {
      if (err instanceof ProviderOverloadedError) return 5 * 60_000;
      const base = Math.min(30_000 * 2 ** (attempts - 1), 15 * 60_000);
      return base + Math.floor(Math.random() * 10_000);               // jitter
    },
  },
});
```

### 4.4 Timeouts, heartbeat, stale-run recovery

| Cơ chế | Giá trị | Ghi chú |
|---|---|---|
| BullMQ `lockDuration` / `lockRenewTime` | 60s / 20s | Worker tự renew khi process còn sống |
| BullMQ `stalledInterval` / `maxStalledCount` | 30s / 2 | Worker crash → job về `waiting` → attempt mới |
| App heartbeat | `heartbeat:<nodeRunId>` TTL 90s + `NodeRun.heartbeatAt` mỗi 30s | Ghi kèm `lastLogLine` để UI thấy agent còn sống |
| Node timeout | `timeoutMs` theo node | Worker `AbortController` → `SIGTERM` subprocess, sau 15s `SIGKILL` (Windows: `taskkill /PID /T /F`) |
| Approval timeout | `expiresAt` | Cron mỗi phút: `UPDATE "Approval" SET decision='EXPIRED'` → `onTimeout` |
| Reaper | Cron mỗi 60s | `RUNNING AND heartbeatAt < now()-3min` → `FAILED(STALE)` → retry nếu còn attempt; so khớp `SMEMBERS sem:*` với NodeRun RUNNING để xoá slot mồ côi |
| Startup recovery | khi NestJS boot | Mọi run `RUNNING` → re-enqueue `tick`; `QUEUED` nhưng không có BullMQ job → về `READY`; `RUNNING` không heartbeat → như reaper |

**Pause/Resume/Cancel**:
- `pause(run)`: `status=PAUSED`; scheduler bỏ qua; node đang chạy **chạy hết** (không kill, tránh mất công agent) nhưng không dispatch mới. `hard=true` → kill.
- `resume(run)`: `status=RUNNING` + tick.
- `cancel(run)`: `status=CANCELLING`; worker check Redis flag `cancel:<runId>` mỗi 5s và trước khi spawn → kill subprocess; khi không còn RUNNING → `CANCELLED`; approval PENDING → EXPIRED; giải phóng semaphore.
- Global pause (bảo trì): `queue.pause()` cho các queue `node.*`; scheduler vẫn chạy, node đứng ở READY/QUEUED.

### 4.5 Lifecycle state machine (XState v5, dùng để *validate* transition)

```ts
export const nodeRunMachine = setup({}).createMachine({
  id: 'nodeRun', initial: 'PENDING',
  states: {
    PENDING:       { on: { DEPS_OK: 'READY', SKIP: 'SKIPPED', CANCEL: 'CANCELLED' } },
    READY:         { on: { SLOT: 'QUEUED', OPEN_APPROVAL: 'WAITING_HUMAN', CANCEL: 'CANCELLED' } },
    QUEUED:        { on: { START: 'RUNNING', CANCEL: 'CANCELLED', LOST: 'READY' } },
    RUNNING:       { on: { OK: 'SUCCEEDED', ERR: 'FAILED', STALE: 'FAILED', CANCEL: 'CANCELLED' } },
    // REJECT vẫn là SUCCEEDED với output.handle='rejected' — approval node "hoàn thành" việc của nó
    WAITING_HUMAN: { on: { APPROVE: 'SUCCEEDED', REJECT: 'SUCCEEDED', EXPIRE: 'FAILED', CANCEL: 'CANCELLED' } },
    FAILED:        { on: { RETRY: 'READY', REVISE: 'PENDING' } },
    SUCCEEDED:     { on: { REVISE: 'PENDING' } },
    SKIPPED: { type: 'final' }, CANCELLED: { type: 'final' },
  },
});
// NodeRunService.transition(nodeRun, event): snapshot = restore(status) → send(event) → nếu không đổi state → throw IllegalTransition
```

---

## 5. Agent Message Bus

### 5.1 Mục tiêu
Agent↔Agent không chat tự do (đốt token, loop vô hạn). Mọi message là **structured**, đi qua Postgres (`AgentMessage`) để durable, fan-out realtime qua Redis pub/sub, và chỉ được **inject vào prompt của agent nhận** ở lần spawn/attempt kế tiếp hoặc qua "mailbox check" mà agent gọi bằng tool `check_inbox` (MCP tool, phối hợp Agent 05).

### 5.2 Message types & payload

```ts
type TaskBrief = {            // nguyên tắc token-economy: CHỈ các trường này
  task: string;               // ≤ 1200 chars, imperative
  files: { path: string; reason: string }[];          // ≤ 20 path, KHÔNG gửi nội dung file
  decisions: string[];        // ADR-style bullet, ≤ 10
  outputsFrom: { nodeId: string; summary: string }[]; // summary ≤ 400 chars mỗi node
  blockers: string[];
  acceptance?: string[];      // tiêu chí done
};
type AgentMessageType =
  | 'HANDOFF'          // chuyển giao task (PM→dev, dev→QA)
  | 'HELP_REQUEST'     // cần người khác giải đáp/làm hộ phần nhỏ: { question, context ≤ 600 chars }
  | 'HELP_RESPONSE'
  | 'REVIEW_REQUEST'   // kèm branch ref + files, không kèm diff
  | 'REVIEW_RESULT'    // { verdict: 'approve'|'changes', items: RejectItem[] }
  | 'BLOCKER'          // tới PM; PM quyết định: reassign / tạo node mới / hỏi human
  | 'INFO';            // broadcast ngắn ≤ 300 chars
```

### 5.3 Quy tắc token-economy (enforced bởi `MessageBusService.validate`)
1. Không bao giờ chuyển *transcript* hay *stdout* của agent khác; chỉ `output.summary` + `decisions`.
2. `files[]` là path + lý do; agent nhận tự đọc file trong worktree của mình (sau khi rebase nhánh của người gửi — Agent 04/git service lo).
3. `tokenEst = ceil(JSON.length / 4)`; từ chối message > 3000 token; PM có quota 20 message/node.
4. HELP_REQUEST tối đa 2 vòng/thread; vòng 3 tự chuyển thành BLOCKER lên PM.
5. Inbox được *compact* trước khi inject: gom theo `threadId`, giữ message mới nhất mỗi thread, tổng ≤ 4000 token.
6. OpenHands pattern: "child không thấy history của parent — everything it needs must be in the task brief" → tinh thần của `TaskBrief`.

### 5.4 Flow handoff & review động

```
dev_backend (node N1) SUCCEEDED → output {summary, files, decisions}
  → scheduler resolveInput(N2 = qa) áp edge.carry → NodeRun.input = TaskBrief
  → bus.send({type:'HANDOFF', from:'pm', to:qaAgentId, payload: brief})   // cùng transaction
  → event AGENT_MESSAGE_SENT → WS + virtual office (bong bóng "đang bàn giao")
```

`REVIEW_REQUEST` có thể do agent tự gửi (tool `request_review(target_role)`): bus tạo **node_run động** kind `agent` role=reviewer với `nodeId='dyn:review:<ulid>'`, predecessor = node hiện tại; scheduler coi như node bình thường — graph JSON không đổi, chỉ `node_runs` thêm hàng. Kết quả review trở thành `REVIEW_RESULT` gửi lại agent gốc (inject ở attempt kế tiếp nếu agent đã exit, hoặc qua `check_inbox` nếu còn chạy).

---

## 6. PM Agent Auto-Assignment

Khi node `agent` có `assignee='auto'`, `AssignmentService.pick(nodeRun)` chạy **deterministic scoring** (không gọi LLM) — PM Agent (LLM) chỉ can thiệp khi không có candidate hoặc có BLOCKER.

```ts
type Candidate = { id: string; roles: string[]; skills: Record<string, number>; // 0..1
  provider: 'claude'|'codex'; permissions: string[]; activeTasks: number; maxParallel: number;
  recentFailRate: number };

export function scoreCandidate(c: Candidate, node: AgentNodeData,
  ctx: { providerFree: Record<string, number>; requiredSkills: string[]; upstreamAssignees: string[] }) {
  if (!c.roles.includes(node.role)) return -Infinity;                               // hard filter
  if (node.permissions.some(p => !c.permissions.includes(p))) return -Infinity;      // hard filter
  if (node.provider !== 'any' && c.provider !== node.provider) return -Infinity;
  if (c.activeTasks >= c.maxParallel) return -Infinity;

  const skill   = avg(ctx.requiredSkills.map(s => c.skills[s] ?? 0));             // 0..1
  const load    = 1 - c.activeTasks / c.maxParallel;                                // 0..1
  const provAv  = Math.min(1, ctx.providerFree[c.provider] / 3);                    // slot trống của provider
  const context = ctx.upstreamAssignees.includes(c.id) ? 1 : 0;                     // đã làm node trước → có context
  const health  = 1 - c.recentFailRate;
  return 0.35*skill + 0.25*load + 0.15*provAv + 0.15*context + 0.10*health;
}
```

- `requiredSkills` suy từ `node.role` + tags trong `promptTemplate` (vd `#nestjs #prisma`) + đuôi file trong `input.files`.
- `providerFree` đọc từ `SCARD sem:provider:*` so với cap → ưu tiên provider đang rảnh (giảm SEMAPHORE_WAIT).
- Tie-break: ít `activeTasks` hơn → id nhỏ hơn (ổn định, test được).
- Không có candidate (tất cả −∞) → node vẫn READY, phát `ASSIGNMENT_BLOCKED` → spawn PM Agent node động với prompt "chọn 1 trong N hoặc đề xuất tạo employee mới"; PM không giải quyết trong 10 phút → human notification.
- Kết quả ghi `NodeRun.assigneeId`, event `TASK_ASSIGNED { score, breakdown }` để UI giải thích vì sao.

---

## 7. Event Catalogue

### 7.1 Envelope

```ts
interface WorkspaceEvent<T extends EventType = EventType> {
  id: string;            // ulid — ordering + cursor cho replay
  type: T;
  ts: string;            // ISO
  workspaceId: string;
  runId?: string; nodeRunId?: string; agentId?: string;
  actor: { kind: 'system'|'agent'|'human'; id: string };
  payload: EventPayload[T];
  v: 1;
}
```

### 7.2 Danh mục

| Type | Payload chính | Nguồn |
|---|---|---|
| `AGENT_CREATED` / `AGENT_UPDATED` / `AGENT_DELETED` | `{ agent: {id, name, role, provider} }` | API |
| `AGENT_STATUS_CHANGED` | `{ status: 'idle'\|'working'\|'waiting'\|'error'\|'offline', nodeRunId? }` | Worker/Reaper |
| `AGENT_HEARTBEAT` | `{ lastLine, tokensIn, tokensOut, elapsedMs }` | Worker (throttle 5s) |
| `AGENT_OUTPUT_CHUNK` | `{ seq, text }` | Worker stream (chỉ WS, không outbox) |
| `AGENT_MESSAGE_SENT` | `{ messageId, type, from, to, threadId, tokenEst }` | Bus |
| `TASK_ASSIGNED` / `TASK_REASSIGNED` / `ASSIGNMENT_BLOCKED` | `{ nodeRunId, agentId?, score?, breakdown? }` | Assignment |
| `WORKFLOW_STARTED` / `WORKFLOW_PAUSED` / `WORKFLOW_RESUMED` / `WORKFLOW_CANCELLED` / `WORKFLOW_COMPLETED` / `WORKFLOW_FAILED` | `{ runId, definitionVersionId, revision, reason? }` | RunService/Scheduler |
| `WORKFLOW_STAGE_CHANGED` | `{ runId, fromStage, toStage, revision }` | Scheduler (khi `data.stage` của node active đổi) |
| `NODE_READY` / `NODE_QUEUED` / `NODE_STARTED` / `NODE_SUCCEEDED` / `NODE_FAILED` / `NODE_SKIPPED` / `NODE_RETRY_SCHEDULED` | `{ nodeRunId, nodeId, kind, attempt, delayMs?, error? }` | Scheduler/Worker |
| `HUMAN_APPROVAL_REQUESTED` | `{ approvalId, nodeRunId, approverRoles, artifacts, expiresAt }` | ApprovalService |
| `HUMAN_APPROVED` / `HUMAN_REJECTED` / `HUMAN_APPROVAL_EXPIRED` | `{ approvalId, decidedBy, comment, rejectItems? }` | ApprovalService |
| `REVISION_CREATED` | `{ runId, revision, fixTasks: [{nodeId, assigneeId, brief}] }` | PM Agent / RevisionService |
| `CHECKPOINT_SAVED` | `{ runId, seq, reason }` | Scheduler |
| `SEMAPHORE_WAIT` | `{ nodeRunId, provider, globalUsed, providerUsed }` | Dispatcher |
| `WORKTREE_CREATED` / `WORKTREE_MERGED` / `WORKTREE_CONFLICT` | `{ path, branch, nodeRunId }` | Git service (Agent 04) |
| `TEST_RESULT` / `DEPLOYMENT_RESULT` | `{ nodeRunId, passed, summary, reportRef }` | Test/Deploy worker |

### 7.3 Fan-out

```
Transaction (Postgres) ──▶ EventOutbox row
   OutboxRelay (poll 250ms: SELECT … WHERE published=false ORDER BY id LIMIT 500 FOR UPDATE SKIP LOCKED)
      ├─▶ Redis PUBLISH ws:workspace:<id>      → Socket.IO (@socket.io/redis-adapter)
      │        rooms: workspace:<id>, run:<id>, agent:<id>
      └─▶ Redis XADD events:<workspaceId>      → consumers: VirtualOffice projector, analytics, webhook
```

- **Virtual Office projector** (Agent 09) subscribe stream, duy trì `office_state` (vị trí/animation mỗi employee): `AGENT_STATUS_CHANGED.working` → ngồi gõ; `HUMAN_APPROVAL_REQUESTED` → PM đi tới "phòng họp"; `AGENT_MESSAGE_SENT` → bong bóng chat giữa 2 bàn. Projector là *read-only*, rebuild được từ stream (`XRANGE`).
- `AGENT_OUTPUT_CHUNK` không ghi outbox (quá nhiều) — publish thẳng Redis; UI reconnect thì đọc log qua `stdoutRef`.
- Client reconnect gửi `lastEventId` → server replay từ `EventOutbox WHERE id > ?` (giữ 7 ngày, sau đó archive).

---

## 8. Revision Loop (Human Reject)

Nguyên tắc: **không chạy lại toàn bộ workflow**, chỉ các node *bị ảnh hưởng*; PM tạo *fix tasks có mục tiêu* từ `rejectItems`.

```
HUMAN_REJECTED(approval A tại node G, rejectItems[])
 1. Scheduler: Checkpoint(reason='before_revision', seq++)        // để rewind
 2. run.revision += 1 ; event WORKFLOW_STAGE_CHANGED(toStage = stage của G, revision)
 3. Spawn PM Agent node động 'dyn:pm-triage:<ulid>' với input:
      { rejectItems, candidates: agent nodes upstream của G trong cùng stage, outputs.summary của chúng }
    PM trả JSON: fixTasks = [{ targetNodeId, assignee: 'same'|'auto', brief: TaskBrief, scope: 'patch' }]
    (PM không trả JSON hợp lệ sau 2 lần → fallback: fix task cho MỌI agent node upstream trực tiếp của G)
 4. Với mỗi fixTask: INSERT NodeRun { nodeId: targetNodeId, revision: run.revision, status: PENDING→READY,
      input: brief (kèm output cũ làm 'outputsFrom'), assignee: người cũ nếu 'same' (giữ context) }
    Các node trên đường từ targetNode → G (merge/tool/test/review…) cũng INSERT bản revision mới (PENDING).
    Node G (approval) INSERT bản revision mới. Node KHÔNG nằm trên đường đó GIỮ NGUYÊN (SUCCEEDED, revision cũ).
 5. tick(run) → chỉ node revision mới chạy; latestByNodeId() lấy bản mới nhất mỗi nodeId.
 6. Vượt maxRevisions (approval.onReject.maxRevisions) → run FAILED(REVISION_LIMIT) + human notification.
```

**Revision numbering**: `run.revision` toàn cục (`R0, R1, …`); `NodeRun` unique `(runId, nodeId, revision)`; UI hiển thị `Stage 5 · R2`. Worktree của fix task: branch `wf/<runId>/<nodeId>/r<rev>` cắt từ branch của attempt SUCCEEDED trước đó (patch, không làm lại từ `main`). Artifact cho reviewer ở revision mới kèm **diff so với revision trước** để duyệt nhanh.

Rewind thủ công: `POST /runs/:id/rewind?seq=N` → restore snapshot; mọi NodeRun có revision > snapshot.revision → `CANCELLED`; run.revision = snapshot.revision; tick.

---

## 9. Default Template: FULL SOFTWARE DEVELOPMENT LIFECYCLE

8 stage: 1 Requirements → 2 Architecture → 3 Planning (PM) → 4 Implementation (parallel FE/BE/DB) → 5 Integration & Review → 6 QA/Test → 7 Deployment Staging → 8 Release. Approval gate sau stage 1, 2, 5, 6, 7. `position` chỉ để React Flow render.

```json
{
  "schemaVersion": 1,
  "name": "FULL SOFTWARE DEVELOPMENT LIFECYCLE",
  "variables": { "baseBranch": "main", "stagingUrl": "" },
  "nodes": [
    { "id": "start", "type": "start", "position": { "x": 0, "y": 300 }, "data": { "kind": "start", "label": "Start" } },

    { "id": "s1_ba", "type": "agent", "position": { "x": 200, "y": 300 },
      "data": { "kind": "agent", "label": "S1 · Requirements Analysis", "stage": "1-requirements", "role": "business_analyst", "provider": "any",
        "promptTemplate": "Phân tích yêu cầu sau thành PRD (docs/PRD.md): user stories, acceptance criteria, out-of-scope.\n\n{{input.brief}}",
        "outputs": ["summary", "files", "decisions"], "timeoutMs": 1800000 } },
    { "id": "g1", "type": "approval", "position": { "x": 420, "y": 300 },
      "data": { "kind": "approval", "label": "Gate 1 · Approve PRD", "stage": "1-requirements", "approverRoles": ["owner", "product"],
        "reviewArtifacts": ["s1_ba.files"], "timeoutMs": null, "onReject": { "mode": "revision_loop", "maxRevisions": 3 } } },

    { "id": "s2_arch", "type": "agent", "position": { "x": 640, "y": 300 },
      "data": { "kind": "agent", "label": "S2 · Architecture & Tech Design", "stage": "2-architecture", "role": "architect", "provider": "claude",
        "promptTemplate": "Từ PRD, viết docs/ARCHITECTURE.md: module boundary, API contract (OpenAPI), DB schema (Prisma), ADR.",
        "inputs": ["s1_ba.summary", "s1_ba.files"], "timeoutMs": 2700000 } },
    { "id": "g2", "type": "approval", "position": { "x": 860, "y": 300 },
      "data": { "kind": "approval", "label": "Gate 2 · Approve Architecture", "stage": "2-architecture", "approverRoles": ["owner", "tech_lead"],
        "reviewArtifacts": ["s2_arch.files"], "timeoutMs": null } },

    { "id": "s3_pm", "type": "agent", "position": { "x": 1080, "y": 300 },
      "data": { "kind": "agent", "label": "S3 · PM Planning & Task Split", "stage": "3-planning", "role": "pm", "provider": "any",
        "promptTemplate": "Chia việc thành 3 nhóm FE/BE/DB. Trả JSON tasks{fe,be,db}, mỗi task có files, acceptance. Không mô tả lại PRD.",
        "inputs": ["s1_ba.summary", "s2_arch.summary", "s2_arch.decisions"], "outputs": ["summary", "decisions", "tasks"] } },

    { "id": "s4_par", "type": "parallel", "position": { "x": 1300, "y": 300 },
      "data": { "kind": "parallel", "label": "S4 · Fan-out", "stage": "4-implementation", "maxConcurrency": 3 } },
    { "id": "s4_db", "type": "agent", "position": { "x": 1520, "y": 120 },
      "data": { "kind": "agent", "label": "S4 · DB & Migrations", "stage": "4-implementation", "role": "backend_dev", "provider": "any",
        "promptTemplate": "Thực hiện tasks.db: Prisma schema + migrations + seed. Commit vào branch của bạn.",
        "inputs": ["s3_pm.tasks.db", "s2_arch.files"], "worktree": { "base": "{{variables.baseBranch}}", "isolate": true }, "timeoutMs": 2700000 } },
    { "id": "s4_be", "type": "agent", "position": { "x": 1520, "y": 300 },
      "data": { "kind": "agent", "label": "S4 · Backend (NestJS)", "stage": "4-implementation", "role": "backend_dev", "provider": "any",
        "promptTemplate": "Thực hiện tasks.be theo API contract. Unit test cho service.",
        "inputs": ["s3_pm.tasks.be", "s2_arch.files"], "worktree": { "base": "{{variables.baseBranch}}", "isolate": true }, "timeoutMs": 3600000 } },
    { "id": "s4_fe", "type": "agent", "position": { "x": 1520, "y": 480 },
      "data": { "kind": "agent", "label": "S4 · Frontend (React)", "stage": "4-implementation", "role": "frontend_dev", "provider": "any",
        "promptTemplate": "Thực hiện tasks.fe dùng mock theo OpenAPI; không chờ backend.",
        "inputs": ["s3_pm.tasks.fe", "s2_arch.files"], "worktree": { "base": "{{variables.baseBranch}}", "isolate": true }, "timeoutMs": 3600000 } },
    { "id": "s4_merge", "type": "merge", "position": { "x": 1740, "y": 300 },
      "data": { "kind": "merge", "label": "S4 · Join", "stage": "4-implementation", "strategy": "all", "onBranchFailure": "fail" } },

    { "id": "s5_int", "type": "tool", "position": { "x": 1960, "y": 300 },
      "data": { "kind": "tool", "label": "S5 · Git Integrate", "stage": "5-integration", "tool": "git.integrate",
        "args": { "branches": ["s4_db.branch", "s4_be.branch", "s4_fe.branch"], "into": "wf/{{run.id}}/integration" }, "timeoutMs": 600000 } },
    { "id": "s5_conflict", "type": "condition", "position": { "x": 2180, "y": 300 },
      "data": { "kind": "condition", "label": "Conflicts?", "stage": "5-integration",
        "rules": [ { "handle": "conflict", "when": { ">": [ { "var": "s5_int.conflicts" }, 0 ] } } ], "defaultHandle": "clean" } },
    { "id": "s5_fix", "type": "agent", "position": { "x": 2180, "y": 480 },
      "data": { "kind": "agent", "label": "S5 · Resolve Conflicts", "stage": "5-integration", "role": "backend_dev", "provider": "claude",
        "promptTemplate": "Giải quyết conflict trong các file sau, giữ ý định của cả hai nhánh.", "inputs": ["s5_int.conflictFiles"], "timeoutMs": 1800000 } },
    { "id": "s5_review", "type": "agent", "position": { "x": 2400, "y": 300 },
      "data": { "kind": "agent", "label": "S5 · Code Review (AI)", "stage": "5-integration", "role": "reviewer", "provider": "claude",
        "promptTemplate": "Review diff nhánh integration so với base. Trả JSON {verdict, items[]}.",
        "inputs": ["s5_int.branch"], "outputs": ["summary", "verdict", "items"], "timeoutMs": 1800000 } },
    { "id": "g5", "type": "approval", "position": { "x": 2620, "y": 300 },
      "data": { "kind": "approval", "label": "Gate 5 · Approve Code", "stage": "5-integration", "approverRoles": ["tech_lead"],
        "reviewArtifacts": ["s5_review.summary", "s5_int.branch"], "timeoutMs": null, "onReject": { "mode": "revision_loop", "maxRevisions": 3 } } },

    { "id": "s6_test", "type": "test", "position": { "x": 2840, "y": 300 },
      "data": { "kind": "test", "label": "S6 · Automated Tests", "stage": "6-qa", "command": "pnpm test -- --ci", "cwd": ".", "timeoutMs": 1200000 } },
    { "id": "s6_pass", "type": "condition", "position": { "x": 3060, "y": 300 },
      "data": { "kind": "condition", "label": "Tests pass?", "stage": "6-qa",
        "rules": [ { "handle": "fail", "when": { "==": [ { "var": "s6_test.passed" }, false ] } } ], "defaultHandle": "pass" } },
    { "id": "s6_qa_fix", "type": "agent", "position": { "x": 3060, "y": 480 },
      "data": { "kind": "agent", "label": "S6 · Fix failing tests", "stage": "6-qa", "role": "qa", "provider": "any",
        "promptTemplate": "Sửa lỗi từ báo cáo test; không sửa test để pass giả.", "inputs": ["s6_test.summary", "s6_test.reportRef"], "timeoutMs": 1800000 } },
    { "id": "g6", "type": "approval", "position": { "x": 3280, "y": 300 },
      "data": { "kind": "approval", "label": "Gate 6 · QA Sign-off", "stage": "6-qa", "approverRoles": ["qa_lead", "owner"],
        "reviewArtifacts": ["s6_test.reportRef"], "timeoutMs": 172800000, "onTimeout": "escalate" } },

    { "id": "s7_deploy", "type": "deployment", "position": { "x": 3500, "y": 300 },
      "data": { "kind": "deployment", "label": "S7 · Deploy Staging", "stage": "7-staging", "target": "staging", "script": "scripts/deploy-staging.sh", "requiresPermission": "deploy:staging" } },
    { "id": "g7", "type": "approval", "position": { "x": 3720, "y": 300 },
      "data": { "kind": "approval", "label": "Gate 7 · UAT Approve Release", "stage": "7-staging", "approverRoles": ["owner"], "reviewArtifacts": ["s7_deploy.summary"], "timeoutMs": null } },

    { "id": "s8_release", "type": "deployment", "position": { "x": 3940, "y": 300 },
      "data": { "kind": "deployment", "label": "S8 · Release Production", "stage": "8-release", "target": "production", "script": "scripts/release.sh", "requiresPermission": "deploy:production" } },
    { "id": "s8_notes", "type": "agent", "position": { "x": 4160, "y": 300 },
      "data": { "kind": "agent", "label": "S8 · Release Notes", "stage": "8-release", "role": "pm", "provider": "any",
        "promptTemplate": "Viết CHANGELOG từ decisions + summary các stage.", "inputs": ["s3_pm.summary", "s5_review.summary", "s8_release.summary"], "timeoutMs": 600000 } },
    { "id": "end", "type": "end", "position": { "x": 4380, "y": 300 }, "data": { "kind": "end", "label": "Done" } }
  ],
  "edges": [
    { "id": "e0",  "source": "start",    "target": "s1_ba" },
    { "id": "e1",  "source": "s1_ba",    "target": "g1" },
    { "id": "e2",  "source": "g1",       "target": "s2_arch",   "sourceHandle": "approved" },
    { "id": "e3",  "source": "s2_arch",  "target": "g2" },
    { "id": "e4",  "source": "g2",       "target": "s3_pm",     "sourceHandle": "approved" },
    { "id": "e5",  "source": "s3_pm",    "target": "s4_par" },
    { "id": "e6a", "source": "s4_par",   "target": "s4_db" },
    { "id": "e6b", "source": "s4_par",   "target": "s4_be" },
    { "id": "e6c", "source": "s4_par",   "target": "s4_fe" },
    { "id": "e7a", "source": "s4_db",    "target": "s4_merge" },
    { "id": "e7b", "source": "s4_be",    "target": "s4_merge" },
    { "id": "e7c", "source": "s4_fe",    "target": "s4_merge" },
    { "id": "e8",  "source": "s4_merge", "target": "s5_int" },
    { "id": "e9",  "source": "s5_int",   "target": "s5_conflict" },
    { "id": "e10", "source": "s5_conflict", "target": "s5_fix",    "sourceHandle": "conflict" },
    { "id": "e11", "source": "s5_conflict", "target": "s5_review", "sourceHandle": "clean" },
    { "id": "e12", "source": "s5_fix",   "target": "s5_review" },
    { "id": "e13", "source": "s5_review","target": "g5" },
    { "id": "e14", "source": "g5",       "target": "s6_test",   "sourceHandle": "approved" },
    { "id": "e15", "source": "s6_test",  "target": "s6_pass" },
    { "id": "e16", "source": "s6_pass",  "target": "s6_qa_fix", "sourceHandle": "fail" },
    { "id": "e17", "source": "s6_qa_fix","target": "s6_test",   "data": { "loopBack": true } },
    { "id": "e18", "source": "s6_pass",  "target": "g6",        "sourceHandle": "pass" },
    { "id": "e19", "source": "g6",       "target": "s7_deploy", "sourceHandle": "approved" },
    { "id": "e20", "source": "s7_deploy","target": "g7" },
    { "id": "e21", "source": "g7",       "target": "s8_release","sourceHandle": "approved" },
    { "id": "e22", "source": "s8_release","target": "s8_notes" },
    { "id": "e23", "source": "s8_notes", "target": "end" }
  ]
}
```

Ghi chú về template:
- `s5_review` có 2 incoming edges nhưng chỉ *một* active mỗi lần (condition chọn handle). Với rule "≥1 edge active, không edge pending, inactive → bỏ qua" ở §4.1, khi "clean" thì `s5_fix` SKIPPED và `s5_review` vẫn ready. Đây là semantic "OR trên các nhánh exclusive" tương đương `is_triggered()` của ChatDev.
- Loop `s6_qa_fix → s6_test` (`loopBack=true`) có `maxIterations` mặc định 3; vượt → `g6` nhận `rejected` tự động với `rejectItems` từ test report → revision loop §8.
- Reject ở `g5`/`g1` kích hoạt revision loop; PM tạo fix task cho `s4_*` hoặc `s1_ba` tương ứng, **không** chạy lại `s2_arch`/`s3_pm`.
- Gate 6 có timeout 48h với `escalate` (gửi notification lên owner), các gate khác chờ vô hạn — minh họa R3.

---

## 10. Kế hoạch triển khai & rủi ro

**Modules NestJS** (`apps/api/src/orchestration/`): `definition/` (zod, validate, version), `run/` (RunService, XState guard), `scheduler/` (tick, ready-set, dispatcher, Lua semaphore), `workers/` (agent, tool, test, deploy — `@nestjs/bullmq` `WorkerHost`; test/deploy dùng **sandboxed processor** file riêng để crash/CPU-bound không kéo API process và không làm stall lock), `approval/`, `revision/`, `bus/`, `assignment/`, `events/` (outbox relay, WS gateway), `recovery/` (startup + reaper cron).

**Thứ tự build**: (1) schema + definition validate + template import → (2) scheduler + inline nodes (start/end/condition/parallel/merge) với tool node giả → (3) agent worker + heartbeat + semaphore → (4) approval + revision loop → (5) bus + assignment → (6) events/WS → (7) recovery tests (kill -9 worker giữa chừng, restart Redis, restart Postgres, 2 replica scheduler cùng lúc).

**Rủi ro & đối sách**:

| Rủi ro | Đối sách |
|---|---|
| Windows: `SIGTERM` không hoạt động với child tree của CLI | `tree-kill` / `taskkill /PID /T /F`; spawn với `windowsHide`; test riêng trên Win11 |
| Semaphore leak khi worker chết | lease TTL + reaper so khớp `SMEMBERS sem:*` với `NodeRun RUNNING` |
| Hai scheduler tick cùng run | `SELECT FOR UPDATE` + BullMQ `jobId` dedupe |
| Agent output quá lớn làm phình `NodeRun.output` | `output` giới hạn 32KB (summary/decisions/files), stdout ra file (`stdoutRef`) |
| Template edit làm hỏng run đang chạy | run pin `definitionVersionId`; version mới chỉ áp cho run mới |
| Provider rate-limit đồng loạt | queue per provider + `Worker.RateLimitError` + per-provider cap < global |
| Clock skew giữa replica | mọi timeout tính theo `now()` của Postgres |
| JSONLogic/condition truy cập dữ liệu lạ | context cho condition chỉ gồm `output` của upstream + `variables`, không có env |

---

## 11. Nguồn tham khảo

- ChatDev 2.0 source: `workflow/graph.py` (`_execute_node`, pseudo-edge, cancel event), `workflow/topology_builder.py` (Tarjan SCC → super-node → layers), `workflow/executor/{dag,parallel,cycle}_executor.py`, `workflow/executor/resource_manager.py` (semaphore per node type, in-process), `server/services/session_execution.py` (`wait_for_human_input`, Future + 1800s timeout), `entity/configs/node/node.py` (`is_triggered`/`reset_triggers`), `yaml_template/design.yaml`.
- OpenHands frontend: `src/types/agent-server/core/openhands-event.ts` (union event kinds), `src/types/action-type.tsx` (DELEGATE / REJECT / CHANGE_AGENT_STATE / TASK_TRACKING), `src/api/launch-child-conversation-client-tool.ts` (child conversation brief, worktree isolation), `src/types/agent-server/core/base/common.ts` (`ExecutionStatus`: idle/running/paused/waiting_for_confirmation/finished/error/stuck).
- LangGraph JS interrupts: https://docs.langchain.com/oss/javascript/langgraph/interrupts ; reference `interrupt`: https://reference.langchain.com/javascript/langchain-langgraph/index/interrupt ; HITL: https://docs.langchain.com/oss/javascript/langgraph/human-in-the-loop
- BullMQ Flows: https://docs.bullmq.io/guide/flows ; Rate limiting: https://docs.bullmq.io/guide/rate-limiting ; Stalled jobs: https://docs.bullmq.io/guide/workers/stalled-jobs
- Temporal approval pattern: https://docs.temporal.io/design-patterns/approval ; self-hosted guide: https://docs.temporal.io/self-hosted-guide
- Trigger.dev vs Inngest (secondary, cần verify license): https://automationatlas.io/guides/trigger-dev-vs-inngest-2026-comparison/ ; https://hookdeck.com/webhooks/platforms/inngest-alternatives
- XState v5 persistence: https://stately.ai/docs/persistence ; https://stately.ai/blog/2023-10-02-persisting-state
