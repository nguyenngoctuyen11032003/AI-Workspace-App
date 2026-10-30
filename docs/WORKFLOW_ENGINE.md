# WORKFLOW_ENGINE.md — Workflow engine "Conductor": định nghĩa, node, state machine, scheduler tick, template mặc định

Tác giả: Multi-Agent Architect — Phase 0 (chỉ tài liệu). Ngày: 2026-10-30. Trạng thái: DRAFT chờ owner phê duyệt.
Tổng hợp từ: `DECISION_LOG.md` (D-003, D-009, D-010, D-012, D-014), `research/ORCHESTRATION_WORKFLOW_RESEARCH.md` (A07, đặc biệt §2-4, §8, §9), `research/CHATDEV_ANALYSIS.md` (§8, §10), `DATABASE_DESIGN.md` (§3, §4.6, §4.9), `product/FUNCTIONAL_REQUIREMENTS.md` (FR-101..135, FR-199), `architecture/AGENT_SCHEDULER.md`.
Anh em: `AGENT_ARCHITECTURE.md`, `MULTI_AGENT_ORCHESTRATION.md`, `EVENT_SYSTEM.md`. Không chép lại toàn bộ zod/Prisma/Lua của A07; tài liệu này chốt **hợp đồng** (schema, state, quy tắc) và các chỗ lệch với DB/FR.

Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.

> **[OWR-2 Q-20 / D-003 — ĐỊNH HƯỚNG ĐƯỢC DUYỆT, CHƯA KHÓA IMPLEMENTATION CHO TỚI KHI PoC ĐẠT]** Owner chấp thuận hướng BullMQ + PostgreSQL, nhưng chưa khóa implementation cho đến khi PoC chứng minh: crash recovery; idempotency; không chạy lặp node ngoài ý muốn (no duplicate node runs); retry an toàn; trạng thái approval lưu bền vững; worker không bị giữ trong lúc chờ Owner approval. Nếu PoC không đạt, báo cáo phương án thay thế **Temporal** hoặc **LangGraph** để Owner quyết định (WE-01 "không dùng LangGraph/Temporal" chỉ còn hiệu lực trong chừng mực PoC đạt).
> **[OWR-2 Q-11]** 9 loại node UI và 11 engine kind (xem §3). **[OWR-2 Q-05]** thời hạn chờ approval: xem §7.

---

## 1. Mục đích

Định nghĩa engine thực thi workflow tự xây ("Conductor", D-003): DAG executor trên BullMQ với trạng thái run lưu Postgres; approval là trạng thái `WAITING_HUMAN` không giữ worker. Tài liệu chốt: JSON định nghĩa tương thích React Flow, các node kind và cấu hình, state machine của node_run/run, vòng tick của scheduler, cơ chế chờ approval, versioning, luật validate, template mặc định "FULL SOFTWARE DEVELOPMENT LIFECYCLE", và ánh xạ khái niệm ChatDev 1.0/2.0.

## 2. Phạm vi

- Trong: định nghĩa, node kinds, state machine, tick, approval waiting, versioning, validation, template, ChatDev mapping.
- Ngoài: Workflow Builder UI (FR-112, P2), thuật toán scoring (`AGENT_SCHEDULER.md`), event schema chi tiết (`EVENT_SYSTEM.md`), cơ chế semaphore (`RESOURCE_MANAGEMENT.md`), lifecycle agent (`AGENT_ARCHITECTURE.md`).

## 3. Định nghĩa workflow (JSON, tương thích React Flow)

### 3.1 Nguyên tắc [Đề xuất]

1. **Interchange format** là JSON React Flow: `nodes[] {id, type, position, data}`, `edges[] {id, source, target, sourceHandle, targetHandle, type, label, data}`, `viewport`. Engine **chỉ đọc `data`** và các trường cấu trúc; `position`, `width`, `height` chỉ để render.
2. **Lưu trữ chuẩn** là các bảng đã thiết kế ở `DATABASE_DESIGN.md` §4.6 (`workflows`, `workflow_versions`, `workflow_nodes`, `workflow_edges`: mỗi node/edge một dòng, `node_key`/`edge_key` ổn định giữa các version). A07 lưu cả đồ thị vào một cột `graph Json` (+ `graphHash`). Hợp nhất: **dòng bảng là nguồn**, JSON là định dạng import/export và để tính `graph_hash = sha256(canonical JSON)` phát hiện lưu không đổi; engine nạp thành đồ thị biên dịch trong bộ nhớ, cache theo `workflow_version_id` (xem M-W1).
3. Validate bằng zod trong `packages/contracts` (cùng nơi với event schema, `DATABASE_DESIGN.md` §5.1).
4. Unknown field trong `data` bị **bảo toàn nhưng bỏ qua** (tương thích tiến); `schemaVersion` tăng khi đổi cấu trúc.

### 3.2 Khung tài liệu

```json
{
  "schemaVersion": 1,
  "name": "FULL SOFTWARE DEVELOPMENT LIFECYCLE",
  "description": "...",
  "viewport": { "x": 0, "y": 0, "zoom": 1 },
  "variablesSchema": { "type": "object", "properties": { "baseBranch": { "type": "string", "default": "main" } } },
  "variables": { "baseBranch": "main" },
  "settings": { "defaultMaxIterations": 3, "defaultRetry": { "maxAttempts": 3, "backoff": "exponential", "baseDelayMs": 30000, "maxDelayMs": 900000 } },
  "nodes": [
    { "id": "s2_ba", "type": "agent", "position": { "x": 200, "y": 300 },
      "data": { "kind": "agent", "label": "S2 · Business Analysis", "stage": "2-business-analysis", "...": "xem §4" } }
  ],
  "edges": [
    { "id": "e2", "source": "s2_ba", "target": "g2_ba", "sourceHandle": null, "targetHandle": null,
      "data": { "carry": ["*"], "loopBack": false } }
  ]
}
```

Quy ước handle: `approved`/`rejected` (approval), tên handle khai báo trong `rules[].handle` + `defaultHandle` (condition), `null` (các node còn lại). `edge.data`: `carry` (output nào đi xuống, mặc định `['*']`), `loopBack` (cạnh quay lại), `maxIterations` (ghi đè mặc định 3).

Bảng ánh xạ `type` giữa A07 và DB: DB (`workflow_nodes.type`) ghi comment `start|agent_task|approval|condition|parallel|join|end`, A07 dùng `agent` và `merge`, FR-113 liệt kê 8 loại (Agent, Approval, Condition, Parallel, Merge, Tool, Test, Deployment). **Chuẩn mới (Đề xuất)**: `start`, `end`, `agent`, `task`, `approval`, `condition`, `parallel`, `merge`, `tool`, `test`, `deployment` (9 kind có cấu hình + start/end). Cột `type` là `text` nên không cần migration; chỉ cần sửa comment. `task` là kind bổ sung so với FR-113 (xem M-W2). **[OWR-2 Q-11 ĐÃ DUYỆT]**: **9 UI node types** (loại người dùng kéo thả/cấu hình trên canvas; suy ra = 9 kind có cấu hình: `agent`, `task`, `approval`, `condition`, `parallel`, `merge`, `tool`, `test`, `deployment`) và **11 engine kinds** (9 kind trên + `start` + `end`). Ánh xạ cụ thể 9 UI types là suy luận từ tài liệu này, chưa được Owner liệt kê: cần Owner xác nhận; FR-113 phải đồng bộ (thuộc agent khác).

## 4. Node kinds và trường cấu hình

Trường chung mọi node (trong `data`): `kind`, `label`, `stage` (nhãn stage cho `WORKFLOW_STAGE_*`), `description?`, `timeoutMs?`, `retry?` (`maxAttempts` 1..10, `backoff`, `baseDelayMs`, `maxDelayMs`, `retryOn[]` trong `TIMEOUT|PROVIDER_ERROR|RATE_LIMIT|NONZERO_EXIT|EVIDENCE_MISSING`). Mặc định lấy từ `settings.defaultRetry`.

### 4.1 `agent` — chạy một agent trên một nhiệm vụ

| Trường | Kiểu | Mặc định | Ý nghĩa |
|---|---|---|---|
| `role` | string | bắt buộc | Mã template role (`pm`, `ba`, `backend`, …) |
| `assignee` | `'auto'` \| agentId | `auto` | AUTO dùng scoring; MANUAL chỉ định instance |
| `provider` | `claude`\|`codex`\|`openhands`\|`any` | `any` | Ràng buộc provider |
| `promptTemplate` | string (handlebars) | bắt buộc | Có biến `input.*`, `variables.*`, `run.id` |
| `inputs` | string[] | [] | Khóa `<nodeId>.<key>` lấy từ upstream (áp `edge.carry`) |
| `outputs` | string[] | `summary, files, decisions` | Khóa output (giới hạn 32 KB; stdout ra file) |
| `acceptanceCriteria` | string[] | bắt buộc ≥ 1 | Tiêu chí nghiệm thu, đầu vào của quy tắc evidence |
| `evidence` | `{ required: ('diff'\|'test_result'\|'review'\|'approval'\|'document')[] }` | theo role | Loại bằng chứng tối thiểu (`MULTI_AGENT_ORCHESTRATION.md` §8.2) |
| `writePatterns` / `readPatterns` | glob[] | từ template role | Manifest ownership (D-009) |
| `worktree` | `{ base, isolate }` | `{ variables.baseBranch, true }` | Branch `agent/<slug>/<task-id>` |
| `permissions` | string[] | [] | Ví dụ `git:push`, `deploy:staging` |
| `timeoutMs` | int | 2 700 000 | Hard kill; mặc định theo complexity ở `RESOURCE_MANAGEMENT.md` §7 |

### 4.2 `task` — đơn vị WBS, có thể mở rộng động [Đề xuất]

Khác `agent` (một lệnh gọi agent cố định), `task` là **vỏ công việc có theo dõi** (một bản ghi `tasks` đầy đủ trường FR-107) và có thể **mở rộng thành nhiều node_run/task động** theo kế hoạch của PM.

| Trường | Kiểu | Mặc định | Ý nghĩa |
|---|---|---|---|
| `taskTemplate` | `{ name, description, acceptanceCriteria[], priority }` | bắt buộc | Dùng khi `expansion.mode='static'` |
| `expansion.mode` | `static`\|`pm_wbs` | `static` | `pm_wbs`: PM agent trả `proposedTasks[]`, validator tạo task + `task_dependencies` |
| `expansion.maxTasks` | int | 60 | Chặn spawn tax (`validateWbs`) |
| `expansion.requiredRoles` | string[] | [] | Giới hạn role được phép trong WBS |
| `assignment` | `{ mode: 'AUTO'\|'MANUAL', agentId? }` | AUTO | Áp cho các task sinh ra |
| `joinPolicy` | `all`\|`any` | `all` | Khi nào node `task` coi là xong |

Node động có `nodeKey='dyn:<ulid>'`, chỉ thêm hàng `node_runs`, **không đổi graph JSON** (A07 §5.4). Cha chỉ `SUCCEEDED` khi các con `COMPLETED` và evidence cấp cha đủ.

### 4.3 `approval` — cổng chờ người

| Trường | Kiểu | Mặc định | Ý nghĩa |
|---|---|---|---|
| `approverRoles` | string[] ≥ 1 | bắt buộc | Nhãn quyền; phải ánh xạ được sang `org_role` (M-W5) |
| `policyId` | uuid? | null | Liên kết `approval_policies` |
| `reviewArtifacts` | string[] | [] | Khóa upstream hiển thị cho reviewer |
| `mandatory` | bool | false | `true` → validator cấm xóa/vô hiệu hóa (FR-199) |
| `timeoutMs` | int\|null | null | ~~null = chờ vô hạn (FR-106)~~ **[OWR-2 Q-05]** Chờ trong tiến trình tối đa **540 000 ms (540 s)**; hết hạn thì lưu trạng thái và chuyển `WAITING_HUMAN`, không giữ worker; `null` không còn nghĩa "giữ worker vô hạn" mà nghĩa "`WAITING_HUMAN` bền vững, nhắc Owner sau 24 giờ". Xem §7 |
| `onTimeout` | `fail`\|`auto_approve`\|`escalate`\|`reject` | `fail` | `auto_approve` bị cấm cho gate `mandatory` |
| `onReject` | `{ mode: 'revision_loop'\|'fail', maxRevisions, target: 'pm_triage'\|'fixed', fixedTargets[] }` | `revision_loop`, 3, `pm_triage` | Xem `MULTI_AGENT_ORCHESTRATION.md` §9 |
| `collect` | `('comment'\|'rejectItems'\|'screenshotMarkup')[]` | `comment, rejectItems` | FR-127, FR-130 |

### 4.4 `condition` — rẽ nhánh

`rules[] { handle, when }` với `when` là **JSONLogic** (không eval JS), `defaultHandle` (mặc định `else`). Context của biểu thức chỉ gồm `output` của upstream và `variables`, **không có env/secret** (A07 §10). Mọi `handle` phải có cạnh ra (validator).

### 4.5 `parallel` và 4.6 `merge`

`parallel`: `maxConcurrency` (mặc định 3; vẫn chịu giới hạn global/provider). `merge`: `strategy` (`all`\|`any`\|`quorum`), `quorum?`, `onBranchFailure` (`fail`\|`continue`). Mỗi `parallel` phải có đúng một `merge` dominator (V08). `merge` chỉ đếm các nhánh thuộc nhóm parallel của nó.

### 4.7 `tool`

`tool` (tên trong registry, ví dụ `git.integrate`, `docs.lint`), `args` (JSON, hỗ trợ template), `runIn` (`docker`\|`host`, mặc định `docker`, D-007), `timeoutMs` (10 phút), `retry`. Tool chạy qua worker riêng; `git.*` luôn qua `GitService`, **không** có tool tự merge vào `main` (D-009).

### 4.8 `test`

`command`, `cwd`, `runIn` (`docker`), `passIfExitZero` (true), `report` (`{ format: 'junit'\|'json'\|'none', path }`), `timeoutMs` (20 phút), `retry` (mặc định 1 attempt). **Ngữ nghĩa**: test chạy xong với kết quả sai vẫn là `SUCCEEDED` với `output.passed=false` (kết quả hợp lệ để `condition` rẽ nhánh); `FAILED` chỉ khi hạ tầng lỗi (timeout, không chạy được). Output: `passed`, `exitCode`, `summary`, `reportRef` (artifact `test_result`).

### 4.9 `deployment`

`target` (`preview`\|`staging`\|`production`), `script`, `requiresPermission` (`deploy:preview` …), `healthCheck` (`{ url, timeoutMs }`), `requiresApprovalFrom` (nodeKey của approval upstream, bắt buộc khi `target='production'`). FR-126: giai đoạn Build & Deploy **không** tự deploy production; V12 cấm `production` nếu không có approval gate `mandatory` ngay trước.

`start`/`end`: chỉ `label`. Đúng 1 `start`, ≥ 1 `end`. Chúng chạy **inline** (< 50 ms) trong tiến trình API cùng `condition`, `parallel`, `merge` (A07 §4.2), không tốn slot.

## 5. State machine

### 5.1 node_run [Đề xuất, giữ nguyên 9 trạng thái của A07]

Enum DB hiện dùng chung `run_status` (7 giá trị: pending/running/waiting/paused/succeeded/failed/cancelled) cho cả `workflow_runs` và `node_runs`; **không đủ** cho `READY`, `QUEUED`, `SKIPPED`, `WAITING_HUMAN` (M-W3). Cần enum mới `node_run_status` (migration). Tên viết HOA trong tài liệu; casing lưu trữ chờ owner (DECISION_LOG mâu thuẫn 5).

| Từ | Đến | Trigger | Guard | Event |
|---|---|---|---|---|
| PENDING | READY | `DEPS_OK` (scheduler) | Mọi incoming edge không-loopBack terminal và ≥1 active | `WORKFLOW_NODE_READY` |
| PENDING | SKIPPED | `SKIP` | Mọi incoming edge inactive | `WORKFLOW_NODE_SKIPPED` |
| READY | QUEUED | `SLOT` | Acquire semaphore thành công (hoặc node inline) | `WORKFLOW_NODE_QUEUED` |
| READY | WAITING_HUMAN | `OPEN_APPROVAL` | `kind='approval'` (không tốn slot) | `HUMAN_APPROVAL_REQUESTED` |
| READY | READY | `NO_SLOT` | Hết slot; ghi `queuedReason` | `RESOURCE_SEMAPHORE_WAIT` |
| QUEUED | RUNNING | `START` | Worker nhận job; session `starting→active` | `WORKFLOW_NODE_STARTED` |
| QUEUED | READY | `LOST` | Không có BullMQ job tương ứng (recovery) | — |
| RUNNING | SUCCEEDED | `OK` | **Evidence guard đạt** (agent/task/test/deployment; xem `MULTI_AGENT_ORCHESTRATION.md` §8) | `WORKFLOW_NODE_SUCCEEDED` |
| RUNNING | FAILED | `ERR`/`STALE`/`TIMEOUT`/`EVIDENCE_MISSING` | Ghi `failureClass` | `WORKFLOW_NODE_FAILED` |
| RUNNING | WAITING_HUMAN | `NEED_APPROVAL` | Lệnh L4 hoặc policy ở giữa node | `HUMAN_APPROVAL_REQUESTED` |
| WAITING_HUMAN | RUNNING | `APPROVE_INLINE` | Approval giữa node được duyệt, session resume | `HUMAN_APPROVED` |
| WAITING_HUMAN | SUCCEEDED | `APPROVE`/`REJECT` | **Node approval**: quyết định người *là* kết quả; `output.handle = approved\|rejected` | `HUMAN_APPROVED`/`HUMAN_REJECTED` |
| WAITING_HUMAN | FAILED | `EXPIRE` | `onTimeout='fail'` | `HUMAN_APPROVAL_EXPIRED` |
| FAILED | READY | `RETRY` | `attempt < maxAttempts` và `failureClass` retryable; chờ backoff | `WORKFLOW_NODE_RETRY_SCHEDULED` |
| FAILED/SUCCEEDED | PENDING | `REVISE` | RevisionService tạo bản revision mới | `WORKFLOW_REVISION_STARTED` |
| PENDING/READY/QUEUED/RUNNING/WAITING_HUMAN | CANCELLED | `CANCEL` | Run bị hủy hoặc node bị bỏ khỏi đường revision | — |
| SKIPPED, CANCELLED | (final) | — | — | — |

Ghi chú: approval reject vẫn là `SUCCEEDED` (node "hoàn thành việc của nó"), nhánh chọn bằng `output.handle`; điều này khớp A07 §4.5. Hợp lệ hóa chuyển trạng thái bằng bảng này hoặc XState v5 (A07 §4.5); sai → `IllegalTransition`.

### 5.2 workflow_run

| State | Ý nghĩa | Vào/ra |
|---|---|---|
| PENDING | Mới tạo | → RUNNING khi start |
| RUNNING | Có node đang chạy/sẵn sàng | → PAUSED, WAITING_HUMAN, COMPLETED, FAILED, CANCELLING |
| WAITING_HUMAN | Mọi node hoạt động đều chờ người (suy ra, không lưu riêng nếu muốn) | → RUNNING khi có quyết định |
| PAUSED | Không dispatch mới; node đang chạy chạy nốt (`hard=true` thì kill) | → RUNNING (resume) |
| CANCELLING | Đang kill session, hủy approval `pending` | → CANCELLED khi không còn node RUNNING |
| COMPLETED | Mọi node tới `end` terminal, không FAILED chưa xử lý | final |
| FAILED | `REVISION_LIMIT`, `LOOP_LIMIT`, node FAILED hết retry không có nhánh xử lý | final |
| CANCELLED | Người hủy | final |

Tương ứng DB `run_status`: `pending, running, waiting, paused, succeeded, failed, cancelled` (CANCELLING không có; coi là `running` + cờ). Ánh xạ: `WAITING_HUMAN→waiting`, `COMPLETED→succeeded` (M-W3).

## 6. Scheduler tick

### 6.1 Hợp đồng [Đề xuất, rút từ A07 §4]

`tick(runId)` là hàm **idempotent**; BullMQ job `jobId = sched:<runId>` tự dedupe; chạy được nhiều replica nhờ `SELECT ... FOR UPDATE` trên dòng `workflow_runs`.

1. **Khóa run** (`FOR UPDATE`); bỏ qua nếu run không ở `RUNNING|WAITING_HUMAN`.
2. Nạp đồ thị biên dịch theo `workflow_version_id` (cache) và `node_runs`; lấy bản **mới nhất mỗi `node_key`** (theo revision/attempt).
3. **Ready-set**: node `PENDING` có `predecessorsSatisfied`: mọi incoming edge không-loopBack ở trạng thái terminal; edge **active** nếu source `SUCCEEDED` và (`sourceHandle` null hoặc khớp `output.handle`); node ready khi ≥1 edge active và không edge nào còn pending; mọi edge inactive → `SKIPPED` lan truyền (tương đương `is_triggered()` của ChatDev 2.0).
4. **Merge**: theo `strategy`; `onBranchFailure='fail'` → merge `FAILED` khi một nhánh hết attempt.
5. **loopBack**: không tính vào ready. Khi source của cạnh loopBack `SUCCEEDED`, reset target và downstream tới đích về `PENDING` với revision/attempt+1; giới hạn `maxIterations` (mặc định 3, M-W6). Hết hạn mức: phát `WORKFLOW_LOOP_LIMIT_REACHED`, node tạo approval "escalation" (mặc định) hoặc fail theo `onLoopLimit`.
6. Đổi `ready` → ghi `input` (áp `edge.carry` + `node.inputs`) và phát `WORKFLOW_NODE_READY`; lan truyền skip; cập nhật `currentStage` và phát `WORKFLOW_STAGE_STARTED/COMPLETED` khi `data.stage` của các node active đổi; tạo checkpoint khi cần.
7. Nếu mọi node terminal: kết thúc run (`COMPLETED`/`FAILED`).
8. **Sau khi commit** (ngoài transaction): `dispatchReady(runId)`: node inline chạy ngay rồi enqueue tick; `approval` mở `approvals` và không tốn slot; node tốn tài nguyên → `AssignmentService.pick` → acquire semaphore đa tầng → `QUEUED` + enqueue vào queue theo kind/provider (`node.agent.claude`, `node.agent.codex`, `node.tool`, `node.test`, `node.deploy`; DB §10 liệt kê `agent-runner`, `workflow-engine` ở mức khái quát, cần thống nhất tên, M-W4).

Mọi chuyển trạng thái = **một transaction + outbox event** (D-014); không "emit rồi mới UPDATE".

### 6.2 Kích hoạt tick

Event-driven: node kết thúc, approval quyết định, semaphore nhả (tick cho các run có node `READY`, giới hạn 50 run/lần), run start/resume. Dự phòng: tick định kỳ 1-2 s cho run `RUNNING` (phòng mất job), và quét khi boot (xem `MULTI_AGENT_ORCHESTRATION.md` §10).

### 6.3 Pause/resume/cancel

`pause`: không dispatch mới, node đang chạy chạy nốt. `resume`: `RUNNING` + tick. `cancel`: `CANCELLING`, cờ Redis `cancel:<runId>` kiểm mỗi 5 s và trước spawn, kill cây process (`taskkill /T /F` trên Windows), approval `pending` → `cancelled`, nhả semaphore. Global pause bảo trì: `queue.pause()` cho `node.*`.

## 7. Chờ approval

1. Node approval tới `READY` → `ApprovalService.open(nodeRun)`: tạo `approvals` (`type`, `status='pending'`, `node_run_id`, `payload` = snapshot artifact reviewer thấy, `expires_at`), node `WAITING_HUMAN`, run có thể `WAITING_HUMAN`. **Không giữ job, worker, hay slot** (lý do số 1 không dùng job chờ trong processor: lock BullMQ stall sau ~30 s, A07 §2.1).
2. Phát `HUMAN_APPROVAL_REQUESTED` (kèm `approverRoles`, artifact, `expiresAt`); UI Approval Center và Virtual Office phản ứng theo event.
3. Quyết định: `UPDATE approvals SET status=..., decision=..., decided_by_user_id=..., decided_at=now() WHERE id=? AND status='pending' RETURNING *` (chống quyết định đôi); cùng transaction cập nhật node_run (`SUCCEEDED` với `output.handle`), ghi outbox, enqueue tick.
4. `approved` → cạnh `approved` active. `rejected` → nếu có cạnh `rejected`, đi theo đó; nếu không và `onReject.mode='revision_loop'` thì RevisionService xử lý (§9 của `MULTI_AGENT_ORCHESTRATION.md`); `fail` → run `FAILED`.
5. **Hết hạn**: một BullMQ delayed job cho mỗi approval (`approval-expiry`, DB §10) **và** cron quét an toàn mỗi phút (A07 dùng cron thuần); chọn kết hợp cả hai, mọi mốc tính theo `now()` của Postgres (M-W7). `onTimeout='escalate'` gửi notification cho owner và giữ `pending` (gia hạn `expires_at`).
6. Approval không có timeout (`timeoutMs=null`) chờ vô hạn; sau restart vẫn còn nguyên vì là dữ liệu Postgres. **[OWR-2 Q-05 ĐÃ DUYỆT, ghi đè các ý trên khi mâu thuẫn]**: (a) thời gian chờ approval tối đa **540 giây**; hết thời gian thì lưu trạng thái và chuyển `WAITING_HUMAN`, **không giữ worker**; (b) có thể nhắc Owner sau **24 giờ**; (c) **production deploy, data deletion và merge `main` cần approval riêng**, approval cho các hành động này **hết hiệu lực sau 15 phút**. Lưu ý: dòng `expires_at` ở bước 1/5 nay có hai nghĩa tách biệt (nhắc 24 giờ với approval thường; hiệu lực 15 phút với ba hành động nhạy cảm): cần Owner/Architect chốt tên cột (`reminder_at` vs `expires_at`), xem DOC_SYNC_LOG_OWR2_B (Conflicts).
7. Quyền quyết định: người dùng có `org_role` thỏa `approverRoles` (M-W5); mọi quyết định ghi audit (FR-201).

## 8. Versioning

- `workflows` (định danh) → `workflow_versions` (`version`, `is_published`, `published_at`, `variables_schema`, `viewport`) → nodes/edges. Template tích hợp có `is_builtin` bất biến; người dùng **clone** rồi sửa (FR-114).
- Sửa = tạo **version mới** (bản nháp, `is_published=false`); chỉ bản published được dùng để start run. Lưu không đổi (`graph_hash` giống) thì không tạo version.
- `workflow_runs.workflow_version_id` **pin** version (FK `RESTRICT`): sửa template **không ảnh hưởng** run đang chạy. Version đang có run không xóa được.
- `node_key` ổn định giữa các version; node động (`dyn:*`) chỉ nằm ở `node_runs`.
- Migrate run đang chạy sang version mới: **ngoài phạm vi MVP** [Giả định]; muốn đổi thì cancel và start lại, hoặc rewind về checkpoint rồi start run mới từ node đó (A07 §8).
- Diff giữa hai version: so sánh theo `node_key`/`edge_key` (UI P2).
- Checkpoint (`CHECKPOINT_SAVED`): snapshot nén `{ nodeRuns{status, outputDigest}, variables, revision }` cho rewind, audit, clone; **resume sau crash không cần checkpoint**, chỉ cần `node_runs` (A07 §3.2). DB lưu `node_runs.checkpoint` (cursor sự kiện); A07 có bảng `Checkpoint` riêng: giữ cả hai, bảng riêng cho snapshot cấp run (M-W8).

## 9. Luật validate (DefinitionService)

Chạy khi lưu và khi publish. `E` = chặn publish, `W` = cảnh báo.

| ID | Mức | Luật |
|---|---|---|
| V01 | E | Đúng 1 node `start`, ≥ 1 `end`; `node.id` duy nhất; schema zod hợp lệ theo `kind` |
| V02 | E | Mọi node reachable từ `start`; mọi node (trừ `end`) có đường tới một `end` |
| V03 | E | Đồ thị là DAG **ngoại trừ** cạnh `loopBack=true`; mỗi chu trình phải chứa ít nhất một cạnh loopBack |
| V04 | E | Mọi cạnh loopBack trỏ về node **tổ tiên** của source (không nhảy sang nhánh khác) |
| V05 | E | `condition`: mọi `handle` (kể cả `defaultHandle`) có cạnh ra; `when` là JSONLogic hợp lệ, chỉ tham chiếu `output`/`variables` |
| V06 | E | `approval` có cạnh `approved`; `rejected` tùy chọn (thiếu thì dùng revision loop) |
| V07 | E | `parallel` có ≥ 2 cạnh ra; `maxConcurrency ≥ 1` |
| V08 | E | Mỗi `parallel` có đúng một `merge` dominator; `merge` không nhận cạnh từ ngoài nhóm |
| V09 | E | `agent`/`task`: `role` tồn tại trong registry hoặc để `agentId` hợp lệ; `acceptanceCriteria` ≥ 1 |
| V10 | E | `inputs` chỉ tham chiếu node upstream và khóa nằm trong `outputs` của nó |
| V11 | E | `writePatterns` hợp lệ (không `..`, không tuyệt đối), nằm trong ownership của role; hai node cùng `parallel` không có `writePatterns` giao nhau |
| V12 | E | `deployment.target='production'` yêu cầu một `approval` `mandatory=true` ngay upstream (FR-126, FR-199) |
| V13 | E | Gate `mandatory` không bị xóa/đổi `onTimeout` thành `auto_approve` so với template gốc (FR-199) |
| V14 | E | `retry.maxAttempts ≤ 10`; `timeoutMs > 0`; `expansion.maxTasks ≤ 200` |
| V15 | E | `tool`/`test` có `runIn='docker'` trừ khi owner đã chấp nhận fallback host (OI-FR-01) |
| V16 | W | Node `agent` thiếu `evidence.required`; test node không có cạnh tới `condition` kiểm `passed` |
| V17 | W | Một `loopBack` không có `maxIterations` rõ ràng (dùng mặc định 3) |
| V18 | W | Gate có `timeoutMs=null` mà không có kênh notification |
| V19 | E | Số node ≤ 500, cạnh ≤ 1 500 (chặn đồ thị quá lớn, [Giả định] ngưỡng) |
| V20 | E | `variables` đáp ứng `variablesSchema` khi start run |

Dry-run (FR-115, P2): duyệt đồ thị với kết quả giả (không gọi provider) để kiểm đường đi; dùng cùng validator và ready-set.

## 10. Template mặc định: FULL SOFTWARE DEVELOPMENT LIFECYCLE

### 10.1 Căn cứ và khác biệt so với A07 §9 [Đề xuất]

Template theo **8 giai đoạn của FR-119..FR-127** (nguồn owner): 1 Requirement Intake, 2 Business Analysis, 3 System Design, 4 UI/UX Design, 5 Development, 6 QA & Security & Code Review & Product Evaluation, 7 Build & Deployment preview, 8 Human Acceptance. A07 §9 đặt tên 8 stage khác (Requirements, Architecture, Planning, Implementation, Integration & Review, QA/Test, Staging, Release) và gate sau stage 1, 2, 5, 6, 7 kèm deploy production; **không khớp FR-199** (gate bắt buộc ở stage 2, 4, 8; production không tự deploy). Tài liệu này theo FR và dùng cơ chế của A07 (M-W9). Gate bắt buộc (FR-199): **g2, g4, g8**; đánh dấu `mandatory`.

### 10.2 Bảng node

| id | kind | stage | Role / cấu hình chính | Ghi chú / evidence |
|---|---|---|---|---|
| start | start | — | — | — |
| s1_intake | agent | 1 | `pm`: phân tích prompt, tạo project, đề xuất team (FR-120) | Evidence: brief + danh sách role |
| g1_team | approval | 1 | owner xác nhận team (FR-132); `mandatory=false`, `onTimeout=auto_approve` theo policy | [Đề xuất] — [Cần người dùng phê duyệt] |
| s2_ba | agent | 2 | `ba`: PRD/SRS, user story, acceptance criteria | Evidence: document + lint |
| g2_ba | approval | 2 | **mandatory** gate; reviewArtifacts = tài liệu BA | FR-121, FR-199 |
| s3_arch | agent | 3 | `architect`: kiến trúc, ADR, API contract, DB schema | Không gate mặc định (owner có thể thêm) |
| s4_uiux | agent | 4 | `uiux`: design system, wireframe, token | FR-123 |
| g4_uiux | approval | 4 | **mandatory** gate | FR-199 |
| s5_plan | task | 5 | `pm`: `expansion.mode=pm_wbs` → task FE/BE/DB và dependency | Sinh node động |
| s5_par | parallel | 5 | `maxConcurrency=3` | — |
| s5_db | agent | 5 | `db_infra`: schema, migration, seed; worktree riêng | Evidence: diff + test migration |
| s5_be | agent | 5 | `backend`: NestJS, API theo contract | Evidence: diff + unit test |
| s5_fe | agent | 5 | `frontend`: UI theo design và mock | Evidence: diff + test |
| s5_join | merge | 5 | `strategy=all`, `onBranchFailure=fail` | — |
| s5_review | agent | 5 | `reviewer`: code review trước khi merge (FR-124); output `{verdict, items[]}` | Độc lập tác giả |
| c5_verdict | condition | 5 | `verdict=='changes'` → handle `changes`; else `pass` | — |
| s5_fix | agent | 5 | role theo `items` (assignee `auto`), loopBack về `s5_review` | `maxIterations=3` |
| s5_int | tool | 5 | `git.integrate` các branch vào `integration/<project>` (D-009, không vào `main`) | — |
| c5_conflict | condition | 5 | `conflicts>0` → `conflict`; else `clean` | OI-FR-08 |
| s5_resolve | agent | 5 | `backend` (hoặc owner task mới hơn): giải conflict; loopBack về `s5_int` | `maxIterations=3` |
| s6_par | parallel | 6 | `maxConcurrency=4` | — |
| s6_test | test | 6 | `pnpm test -- --ci` trong Docker | `output.passed` |
| s6_qa | agent | 6 | `qa`: kiểm thử thủ công, tạo bug record | FR-125 |
| s6_sec | agent | 6 | `security`: review bảo mật, secret scan | — |
| s6_eval | agent | 6 | `reviewer`: product evaluation so với PRD | — |
| s6_join | merge | 6 | `strategy=all`, `onBranchFailure=continue` (thu đủ lỗi) | — |
| c6_defects | condition | 6 | `s6_test.passed==false` hoặc `bugs.count>0` → `defects`; else `clean` | — |
| s6_bugfix | task | 6 | `pm_wbs`: mỗi bug thành fix task (FR-125); loopBack về `s6_par` | `maxIterations=3` |
| s7_build | tool | 7 | `build` trong Docker | — |
| s7_preview | deployment | 7 | `target=preview`, health check; **không** production (FR-126) | Evidence: `PREVIEW_READY` |
| g8_accept | approval | 8 | **mandatory** Human Acceptance: approve/reject/feedback/screenshot markup | FR-127, FR-130 |
| s8_release | agent | 8 | `release`: acceptance report, release notes | Chỉ chạy sau `approved` |
| end | end | — | — | — |

Production deploy **không** nằm trong template; nếu người dùng thêm node `deployment(target=production)`, V12 buộc phải có gate `mandatory` ngay trước.

### 10.3 Bảng cạnh

| id | source → target | handle / điều kiện | Ghi chú |
|---|---|---|---|
| e00 | start → s1_intake | — | |
| e01 | s1_intake → g1_team | — | |
| e02 | g1_team → s2_ba | `approved` | |
| e03 | s2_ba → g2_ba | — | |
| e04 | g2_ba → s3_arch | `approved` | |
| e05 | s3_arch → s4_uiux | — | |
| e06 | s4_uiux → g4_uiux | — | |
| e07 | g4_uiux → s5_plan | `approved` | |
| e08 | s5_plan → s5_par | — | |
| e09a/b/c | s5_par → s5_db / s5_be / s5_fe | — | Fan-out |
| e10a/b/c | s5_db / s5_be / s5_fe → s5_join | — | Fan-in |
| e11 | s5_join → s5_review | — | |
| e12 | s5_review → c5_verdict | — | |
| e13 | c5_verdict → s5_fix | `changes` | |
| e14 | s5_fix → s5_review | `loopBack` | Tối đa 3 vòng |
| e15 | c5_verdict → s5_int | `pass` | |
| e16 | s5_int → c5_conflict | — | |
| e17 | c5_conflict → s5_resolve | `conflict` | |
| e18 | s5_resolve → s5_int | `loopBack` | Tối đa 3 vòng |
| e19 | c5_conflict → s6_par | `clean` | |
| e20a/b/c/d | s6_par → s6_test / s6_qa / s6_sec / s6_eval | — | Fan-out |
| e21a/b/c/d | các nhánh → s6_join | — | Fan-in |
| e22 | s6_join → c6_defects | — | |
| e23 | c6_defects → s6_bugfix | `defects` | |
| e24 | s6_bugfix → s6_par | `loopBack` | Tối đa 3 vòng |
| e25 | c6_defects → s7_build | `clean` | |
| e26 | s7_build → s7_preview | — | |
| e27 | s7_preview → g8_accept | — | |
| e28 | g8_accept → s8_release | `approved` | |
| e29 | s8_release → end | — | |
| e30 | g8_accept → (RevisionService) | `rejected` | Không có cạnh vật lý; revision loop |

Ghi chú: `s5_int` có hai incoming (`c5_verdict.pass`, `s5_resolve.loopBack`); `s5_review` có hai incoming (`s5_join`, `s5_fix.loopBack`). Theo luật ready-set, cạnh loopBack không chặn lần chạy đầu. `s6_par` tương tự với `s6_bugfix`.

### 10.4 Revision khi reject [tham chiếu `MULTI_AGENT_ORCHESTRATION.md` §9]

| Gate bị reject | Target mặc định của PM triage | Không chạy lại |
|---|---|---|
| g1_team | `s1_intake` (đề xuất team lại) | từ s2 trở đi |
| g2_ba | `s2_ba` | s1 |
| g4_uiux | `s4_uiux` (không đụng `s3_arch`) | s1-s3 |
| g8_accept | Một hoặc nhiều node: `s4_uiux` (lỗi thiết kế), `s5_fe/be/db` (lỗi code), kèm các node đường tới `g8_accept` (`s5_join`, `s5_review`, `s6_*`, `s7_*`) | Mọi node ngoài đường ảnh hưởng |

## 11. Ánh xạ ChatDev 1.0 và 2.0 [Đã xác minh bằng đọc mã nguồn, Đề xuất cách tái hiện]

Giấy phép ChatDev Apache-2.0: tái sử dụng nội dung prompt/ý tưởng có attribution (D-012); **không** dùng mã Python.

### 11.1 ChatDev 1.0 (ChatChain)

| Khái niệm 1.0 | Cơ chế 1.0 | Tương đương trong Conductor |
|---|---|---|
| `ChatChain` chain tuyến tính của phase | `execute_chain()` lặp phase | Chuỗi node cùng `stage` theo thứ tự cạnh |
| `SimplePhase` (một cuộc đối thoại 2 vai) | `Phase.execute()` + `RolePlaying` | Một node `agent` (một session); đối thoại 2 vai **không** tái hiện, thay bằng giao tiếp qua artifact/message bus |
| `ComposedPhase` + `cycleNum` | Lặp phase con tới `break_cycle()` | Vòng `loopBack` + `maxIterations` + `condition` (thay cho `break_cycle` mã hóa) |
| `need_reflect` (CEO↔Counselor) | Phiên tự đánh giá cuối phase | Node `agent` reviewer độc lập (`s5_review`), có evidence rule |
| `recruitments`/`RoleConfig` | Chiêu mộ vai khi `make_recruitment` | Stage 1 dựng team từ 18 role seed; role là dữ liệu (D-016) |
| `ChatEnv.env_dict` (state chung mutable) | Biến dùng chung | **Không** dùng state chung; output node + `edge.carry` + `variables` (immutable theo revision) |
| Điều kiện dừng `<INFO>` | Dòng đặc biệt trong chat | Output có cấu trúc (`verdict`, `passed`) → `condition` JSONLogic |
| WareHouse thư mục | Thư mục output | Git worktree + branch (D-009) |
| `Human` config (`input()`) | Phase `CodeReviewHuman` | Node `approval` |

### 11.2 ChatDev 2.0 (DevAll, YAML)

| Khái niệm 2.0 | Cơ chế 2.0 | Tương đương Conductor | Khác biệt đáng chú ý |
|---|---|---|---|
| Node `agent` | AgentNodeExecutor, 1 lần gọi LLM | `agent` | Của ta là 1 **session CLI** 10-40 phút, không phải 1 lệnh gọi LLM |
| Node `human` | Future + timeout 1 800 s, RAM | `approval` | Ta: `WAITING_HUMAN` bền trong Postgres, không timeout mặc định |
| Node `subgraph` | Nhúng graph con | Chưa hỗ trợ MVP; có `parent_node_key` (group) ở DB để dành | [Giả định] P2 |
| Node `python` | Chạy code Python | `tool`/`test` trong Docker | Ta không chạy code tùy ý trên host (D-007) |
| `passthrough` | Gom/chuyển tiếp | `merge` hoặc `edge.carry` | |
| `literal` | Message cố định | `promptTemplate` | |
| `loop_counter`/`loop_timer` | Chặn cạnh ra tới N lần | `loopBack` + `maxIterations`; timer: `timeoutMs` | Mặc định ChatDev `max_iterations=100`, ta **3** (M-W6) |
| Edge `trigger`, `condition` (keyword/function) | Kích hoạt node đích | `sourceHandle` + `condition` JSONLogic; cạnh active/inactive | Ta không có `trigger=false`; dùng `carry` |
| Edge `carry_data`, `keep_message`, `clear_context`, `process` | Điều khiển message | `edge.data.carry`; `TaskBrief` thay cho lịch sử hội thoại | Ta không truyền transcript (token economy) |
| `dynamic` edge `map`/`tree` | Nhân bản node đích theo đơn vị | `parallel` + `task` `pm_wbs` (fan-out động); `merge` thay `tree` | |
| Cycles: Tarjan SCC → super node | CycleExecutionStrategy | **Không** cho chu trình tự do; chỉ `loopBack` tường minh (V03/V04) | Dễ validate, dễ giải thích |
| `is_majority_voting` | Chạy mọi node song song, lấy đa số | `merge strategy='quorum'` trên nhánh song song | [Giả định] dùng cho review nhiều reviewer |
| Layered topological + ThreadPool | `DagExecutionStrategy` | Ready-set theo tick + semaphore Redis (xuyên replica) | ChatDev giới hạn theo node type trong process |
| Checkpoint/resume | **Không có** (session RAM) | **Có**: `node_runs` bền, checkpoint, recovery | Khác biệt quan trọng nhất |
| Retry node | Chỉ retry lời gọi LLM (tenacity) | Retry node/attempt, backoff, handover | |
| Memory (`simple/file/blackboard/mem0`) | Gắn vào node | `agent_memories` có duyệt (D-013) | Không tự lưu |
| Validate YAML 3 lớp | schema, typed, structure | zod + 20 luật V01-V20 | |
| `ChatDev_v1.yaml` | Tái hiện 1.0 bằng 2.0 | Template mặc định tái hiện chu trình DemandAnalysis → Coding → CodeReview → Test bằng node + loopBack | Template của ta theo FR-119, nhiều gate người |

## 12. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| WE-01 | Tự xây Conductor (D-003), không dùng LangGraph/Temporal | A07 §1-2: R8 graph JSON user-editable, R3 approval nhiều ngày, cap xuyên replica | **Định hướng được duyệt (OWR-2 Q-20), chưa khóa implementation cho tới khi PoC đạt**; PoC thất bại thì báo Temporal/LangGraph để Owner quyết |
| WE-02 | Postgres là nguồn sự thật; BullMQ chỉ transport | Mất Redis không mất state | [Đề xuất] |
| WE-03 | Dòng bảng nodes/edges là lưu trữ chuẩn, JSON là interchange | Khớp `DATABASE_DESIGN.md`, vẫn export React Flow | [Đề xuất] |
| WE-04 | 11 `type` (9 kind cấu hình + start/end); thêm `task` | WBS động cần kind riêng | **Đã duyệt số lượng (OWR-2 Q-11: 9 UI node types, 11 engine kinds)**; ánh xạ cụ thể chờ Owner xác nhận |
| WE-05 | Test sai vẫn `SUCCEEDED` với `passed=false` | Cho condition rẽ nhánh; FAILED dành cho lỗi hạ tầng | [Đề xuất] |
| WE-06 | Approval reject = `SUCCEEDED` + `handle='rejected'` | Node hoàn thành việc của nó (A07) | [Đề xuất] |
| WE-07 | Cấm chu trình tự do; chỉ `loopBack` có giới hạn 3 | Dễ validate; chống vòng vô hạn | [Đề xuất] |
| WE-08 | Template theo FR-119..127, gate bắt buộc g2/g4/g8 | FR là yêu cầu owner | [Đã xác minh] FR, [Đề xuất] ánh xạ node |
| WE-09 | Production deploy ngoài template; V12 ép gate | FR-126, FR-199 | [Đề xuất] |
| WE-10 | Retry ở node/attempt, không resume giữa chừng subprocess | Subprocess không idempotent | [Đề xuất] (A07) |

## 13. Dependencies

`DATABASE_DESIGN.md` (bảng workflow, thêm `node_run_status`, cột revision, `task_evidence`), `MULTI_AGENT_ORCHESTRATION.md` (evidence, revision, recovery), `AGENT_ARCHITECTURE.md` (state, session), `EVENT_SYSTEM.md` (event `WORKFLOW_*`, `HUMAN_*`), `RESOURCE_MANAGEMENT.md` (semaphore, retry), `WORKSPACE_AND_GIT_ISOLATION.md` (`git.integrate`, branch), `SECURITY.md` (Docker, tool allowlist), thư viện JSONLogic, BullMQ (`@nestjs/bullmq`).

## 14. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Conductor chưa có PoC; logic ready-set/loopBack/merge lồng nhau dễ lỗi | Property test trên đồ thị ngẫu nhiên; spike recovery trước |
| Đồ thị chứa `loopBack` lồng nhau làm reset sai node | V04, reset chỉ từ target tới source; test ví dụ template |
| JSONLogic cho phép truy cập dữ liệu lạ | Context cố định chỉ gồm `output` upstream và `variables` |
| Hai nguồn lưu đồ thị (bảng vs JSON) lệch nhau | Một đường ghi (DefinitionService), JSON sinh từ bảng |
| Số node động lớn (`pm_wbs`) làm tick chậm | Ngưỡng `maxTasks`, chỉ nạp node active, index `node_runs(status)` |
| Tên queue/enum chưa thống nhất giữa A07 và DB | Chốt ở §6.1, §5.1 trước khi viết migration |
| Template mặc định dài, nhiều gate gây mệt phê duyệt | Chỉ g2/g4/g8 bắt buộc; g1 theo policy; owner tắt được gate không mandatory |

## 15. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Thêm kind `task` (ngoài 8 loại FR-113) và enum `node_run_status` mới.
2. [Cần người dùng phê duyệt] `g1_team` có tồn tại mặc định không (FR-132 chỉ nói owner sửa được team trước khi chạy).
3. [Cần người dùng phê duyệt] Có thêm gate sau System Design (A07 có, FR-199 không).
4. Subgraph/nhóm node (`parent_node_key`) và dry-run: P2.
5. Điều kiện `merge` quorum cho nhiều reviewer: chưa có yêu cầu FR.
6. Cho phép migrate run sang version mới hay không.
7. Ngưỡng V19 (500 node) là ước lượng, cần benchmark.

### Mâu thuẫn nguồn liên quan

| ID | Mâu thuẫn | Xử lý |
|---|---|---|
| M-W1 | A07: cột `graph Json`; DB: bảng nodes/edges riêng | §3.1 dòng bảng là chuẩn |
| M-W2 | FR-113: 8 loại node; yêu cầu hiện tại: 9 (có `Task`); DB comment `agent_task`/`join` | §3.2: 11 `type` |
| M-W3 | A07: 9 trạng thái node_run, 8 trạng thái run; DB: `run_status` 7 giá trị dùng chung | §5: enum mới, ánh xạ |
| M-W4 | Tên queue: A07 `node.agent.*`; DB `agent-runner`, `workflow-engine` | Thống nhất ở Phase 1 |
| M-W5 | A07 `approverRoles` như `product`, `tech_lead`, `qa_lead`; DB `org_role`: owner/admin/member/viewer | Cần bảng ánh xạ quyền (`USER_ROLES_AND_PERMISSIONS.md` chưa đối chiếu ở tài liệu này) |
| M-W6 | `maxIterations`: A07 hằng 3; ChatDev 100; FR không nêu | 3, cấu hình theo cạnh |
| M-W7 | Hết hạn approval: A07 cron mỗi phút; DB delayed job per approval | Dùng cả hai |
| M-W8 | Checkpoint: A07 bảng `Checkpoint`; DB `node_runs.checkpoint` (cursor) | Giữ cả hai, khác mục đích |
| M-W9 | Template: A07 8 stage + gate 1,2,5,6,7 + deploy production; FR-119..127 8 stage khác + gate 2,4,8 + không auto-deploy production | Theo FR |
| M-W10 | A07 dùng Prisma; D-002 Drizzle | Drizzle (D-002 APPROVED, 2026-10-30) |

## 16. Acceptance Criteria (Phase 1)

- [ ] Zod schema cho 11 kind; import/export round-trip với React Flow không mất dữ liệu (test `nodes/edges/viewport`).
- [ ] 20 luật V01-V20 có test dương và âm; template mặc định publish được; xóa `g2/g4/g8` hoặc đổi `onTimeout=auto_approve` bị từ chối (V13).
- [ ] Test ready-set: diamond (A→B,C→D), exclusive condition (nhánh không chọn → `SKIPPED` lan truyền), merge `all/any/quorum`, loopBack reset đúng node và dừng ở 3 vòng với `WORKFLOW_LOOP_LIMIT_REACHED`.
- [ ] Ma trận chuyển trạng thái node_run §5.1 có test: chuyển không có trong bảng → `IllegalTransition`; mỗi chuyển hợp lệ ghi đúng một event cùng transaction.
- [ ] Approval: sau kill -9 và restart, approval `pending` vẫn còn, quyết định đôi bị chặn (`WHERE status='pending'`), hết hạn xử lý theo `onTimeout`; không có worker nào bị giữ trong lúc chờ.
- [ ] Run pin version: sửa template giữa chừng không đổi hành vi run đang chạy; version có run không xóa được.
- [ ] Hai replica scheduler cùng tick một run: kết quả đúng một lần (FOR UPDATE + jobId).
- [ ] Chạy template mặc định bằng provider giả: đi hết 8 giai đoạn, dừng đúng ở g2, g4, g8; reject g8 tạo revision có mục tiêu và không chạy lại stage 1-3.
- [ ] Test node `passed=false` rẽ vào `defects` và vòng `s6_bugfix` dừng sau 3 lần; production deploy không xuất hiện trong template.
- [ ] Owner đã duyệt các mục [Cần người dùng phê duyệt] ở §15.
