# MULTI_AGENT_ORCHESTRATION.md — Điều phối đa agent: phân vai, đồng thời, WBS, giao việc, hồi phục

Tác giả: Multi-Agent Architect — Phase 0 (chỉ tài liệu). Ngày: 2026-10-30. Trạng thái: DRAFT chờ owner phê duyệt.
Tổng hợp từ: `DECISION_LOG.md` (D-003, D-009, D-010, D-014, D-016), `research/ORCHESTRATION_WORKFLOW_RESEARCH.md` (A07), `architecture/AGENT_SCHEDULER.md`, `AGENT_COLLABORATION.md`, `RESOURCE_MANAGEMENT.md`, `AGENT_ORGANIZATION.md` (A13), `DATABASE_DESIGN.md`, `product/FUNCTIONAL_REQUIREMENTS.md` (FR-101..135, FR-231..245).
Anh em: `AGENT_ARCHITECTURE.md` (lifecycle, session), `WORKFLOW_ENGINE.md` (DAG, node_run), `EVENT_SYSTEM.md` (event). Không lặp lại công thức scoring, Lua semaphore, bảng backoff đã có ở A07/A13; chỉ chốt phân vai, quy tắc và chỗ hai nguồn lệch nhau.

> **[OWR-2 Q-20 / D-003]** Định hướng Conductor (BullMQ + PostgreSQL) **được duyệt, chưa khóa implementation cho tới khi PoC đạt** (crash recovery, idempotency, không chạy lặp node, retry an toàn, approval state bền vững, không giữ worker khi chờ). PoC không đạt: báo Temporal/LangGraph để Owner quyết. **[OWR-2 Q-05]** approval chờ tối đa 540 s rồi `WAITING_HUMAN`, nhắc 24 giờ; production deploy/data deletion/merge main cần approval riêng hết hạn sau 15 phút (chi tiết `WORKFLOW_ENGINE.md` §7). **[OWR-2 Q-04/D-009]** merge vào `integration/*` khi qua gate; merge `main` luôn cần Owner approval, Lead Agent không thay thế. **[OWR-2 Q-08]** không tự failover provider khi hết quota. **[OWR-2 Q-07]** concurrency mặc định 5 không phải xác nhận subscription.

Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**. Các phần 3-11 là nội dung chi tiết; số mục được giữ ổn định vì các tài liệu anh em tham chiếu (§8 = completion rule, §9 = revision, §10 = recovery).

---

## 1. Mục đích

Xác định **ai quyết định cái gì** khi nhiều agent cùng làm một dự án: Chief Orchestrator, PM agent và các service; cách giới hạn đồng thời (mặc định 5, tối đa 18); cách xử lý WBS và phụ thuộc; giao việc AUTO/MANUAL; tái phân bổ khi hỏng; vòng revision khi người dùng reject; quy tắc hoàn thành dựa trên bằng chứng; hồi phục sau restart; và phân biệt rõ điều gì **đã kiểm chứng** với điều chỉ là **đề xuất**.

## 2. Phạm vi

- Trong: phân vai, concurrency, WBS/dependency, assignment, failure handling, revision, evidence rule, recovery, trạng thái xác minh.
- Ngoài: lifecycle từng agent (`AGENT_ARCHITECTURE.md`), cú pháp workflow (`WORKFLOW_ENGINE.md`), schema event (`EVENT_SYSTEM.md`), sandbox (`SECURITY.md`), giao diện.

## 3. Phân vai: Chief Orchestrator, PM agent, Services

### 3.1 Định nghĩa [Đề xuất] — [Cần người dùng phê duyệt]

"Chief Orchestrator" **không xuất hiện** trong D-001..D-016 hay trong 18 role seed (D-016). Tài liệu này dùng thuật ngữ như sau để khớp nguyên tắc có sẵn "PM agent không có quyền hệ thống" (`AGENT_COLLABORATION.md` §9):

| Thành phần | Bản chất | Quyền |
|---|---|---|
| **Chief Orchestrator** | **Vai trò quyền lực tối cao ở mức hệ thống**, hiện thực bằng mã xác định (deterministic): `OrchestratorService` là cửa duy nhất đổi trạng thái run/task/agent, cộng với **Owner (người)** là cấp ghi đè cuối. Không phải LLM. | Quyết định cuối về assignment, retry, reassign, pause/cancel, mở/đóng revision, hạ/nâng concurrency, xác nhận evidence. |
| **PM agent** (role 01) | Một agent LLM như mọi agent; chạy như task thường, có quota/budget. | **Chỉ đề xuất**: phân rã yêu cầu thành WBS, ước lượng complexity/priority, gợi ý `preferredAgentId`, triage khi human reject, viết báo cáo/minutes. Không spawn/kill, không đổi owner, không vượt limit. |
| **Services** | Các module NestJS có trách nhiệm hẹp, chạy mã không-LLM. | Thực thi: xem bảng §3.2. |

Một biến thể tùy chọn: một "Chief Orchestrator Agent" LLM làm lớp hội thoại với owner (diễn giải yêu cầu thành thao tác). Chưa cần cho MVP; nếu thêm, nó vẫn chỉ **đề xuất** thao tác lên OrchestratorService. [Giả định — chưa có yêu cầu FR nào cho phần này]

### 3.2 Bảng phân trách nhiệm [Đề xuất, hợp nhất A07 §6, A13 `AGENT_COLLABORATION` §9, FR-109]

| Việc | Chief Orchestrator (OrchestratorService + Owner) | PM agent (LLM) | Service chuyên trách |
|---|---|---|---|
| Tạo project, dựng team từ 18 role (FR-120, FR-132) | Duyệt/ghi; owner sửa team trước khi chạy | Đề xuất team | `ProjectService` |
| Phân rã WBS, task, dependency | Validate rồi tạo task | **Đề xuất** `proposedTasks[]` | `TaskService`, validator đồ thị |
| Chọn agent cho task (AUTO) | **Quyết định** theo kết quả scoring | Gợi ý, hoặc phá hòa khi hết ứng viên | `AssignmentService` (scoring xác định, không gọi LLM) |
| Chọn agent (MANUAL) | Owner chọn, hệ thống kiểm hard filter | — | `AssignmentService` |
| Tick, ready-set, dispatch node | Sở hữu quy tắc | — | `Scheduler` (Conductor) |
| Cấp slot, budget, host guard, kill tree | Đặt chính sách | — | `ResourceManager` |
| Worktree, ownership manifest, diff gate, integration | Quyết định merge vào `integration/<project>` | Không | `GitService` |
| Spawn/stop CLI, chuẩn hóa event | — | — | `ProviderAdapter` |
| Approval gate, timeout, escalation | Áp policy | — | `ApprovalService` |
| Xác nhận evidence, đóng task | **Quyết định** (qua EvidenceService) | Không tự xác nhận task của mình | `EvidenceService` |
| Triage khi human reject, tạo fix task | Duyệt + tạo | **Đề xuất** `fixTasks[]` | `RevisionService` |
| Phát hiện stale/lost, retry, reallocate | **Quyết định** | Không | `Reaper`, `Scheduler` |
| Báo cáo tiến độ, WORK_PLAN/PROJECT_STATUS | Cung cấp dữ liệu | **Làm** | `MetricsService` |
| Giải thích "vì sao chậm" cho owner | Cung cấp evidence (log, metrics) | **Làm** (ngôn ngữ tự nhiên) | — |
| Phát event, outbox, replay | — | — | `EventService` |
| Phục hồi sau restart | Sở hữu quy trình | — | `RecoveryService` |

### 3.3 Nguyên tắc rút ra [Đề xuất]

1. **Mọi thay đổi trạng thái đi qua một cửa** (OrchestratorService → service chuyên trách → transaction + outbox). LLM không ghi trạng thái.
2. **Lỗi PM không làm sập hệ thống**: PM hết quota/lỗi thì scheduler vẫn chạy task đã có; chỉ mất khả năng phân rã/triage mới (có fallback, xem §9).
3. **Người ghi đè được tất cả, và việc ghi đè có audit** (FR-110 MANUAL, FR-239 pause/resume/stop/retry/reassign).
4. Xung đột với FR-109 ("PM agent chấm điểm ứng viên"): hiện thực theo hướng **PM đề xuất, service chấm điểm xác định** (xem M-O1, §15) vì A07 §6 và A13 §5.3 đều dùng scoring xác định, test được.

## 4. Mô hình đồng thời (concurrency)

### 4.1 Con số và cách đếm [Đã xác minh nguồn yêu cầu, Đề xuất cách đếm]

- Mặc định **5** agent chạy đồng thời toàn cục, cấu hình tối đa **18** (D-010; FR-231/232 là yêu cầu owner).
- "Đang chạy" = session `starting`/`active` (và đang dừng); `waiting`, `paused`, `WAITING_HUMAN` **không** chiếm slot nhưng vẫn giữ workspace lock. Đếm theo agent executing, không theo agent đã tạo.
- Có thể tạo 18 instance, chỉ `max_concurrent_agents` chạy cùng lúc.

### 4.2 Các tầng giới hạn (tất cả phải qua mới spawn)

| Tầng | Nguồn cấu hình | Mặc định | Ghi chú |
|---|---|---|---|
| Global | `ResourceConfig.max_concurrent_agents` | 5 (max 18) | Hiệu lực ở tick kế tiếp; giảm không giết session đang chạy |
| Provider/credential | `provider_configs.max_concurrent` | Xem M-O2 | Tài khoản claude.ai `team` dùng chung → pool chung |
| Project | `per_project_defaults.max_concurrent` + weighted fair share | = global | Work-conserving, không chặn cứng nếu project khác rảnh |
| Agent | `max_concurrent_executions` | 1 | Log/context sạch |
| Host guard | RAM tự do ≥ 2 048 MB, CPU 1 phút ≤ 90% | — | Giảm `effective_concurrency` tối thiểu 1 |

Hết slot: task giữ `READY` với `queuedReason` (không xếp hàng trong BullMQ, tránh stalled/lock; A07 §4.2). Thứ tự ưu tiên: priority hiệu dụng (`AGENT_SCHEDULER.md` §5.2), rồi FIFO.

### 4.3 Con số thực nghiệm và giới hạn thật [Đã xác minh]

Đo ngày 2026-10-30 (i5-11400H, 15.77 GB RAM, Windows 11): 1 process `claude -p` ≈ 235 MB RSS peak, 3 process đồng thời tổng đỉnh 685.6 MB, CPU hệ thống đỉnh 79%, không rate-limit ở 3 process, wall time 3 process = 8.17 s so với 17.9 s tuần tự (`RESOURCE_MANAGEMENT.md` §3). Mỗi spawn ~48k token prompt cố định (spawn tax). **Chưa đo** quy mô 5, 10, 18 hoặc task thật có build/test; mọi số cho các mức đó là ngoại suy [Giả định].

Nút thắt thật là quota subscription dùng chung, sau đó RAM khi task nặng, sau đó chất lượng (xung đột workspace, review không kịp). Điều khoản dịch vụ về nhiều phiên song song **chưa được xác minh** (DECISION_LOG mục mâu thuẫn 2) [Cần người dùng phê duyệt].

### 4.4 Quy tắc nâng giới hạn 5 → 10 → 18 [Đề xuất, hiện thực D-010 "chỉ nâng khi có bằng chứng"]

| Nâng lên | Điều kiện bằng chứng (đo trong ≥ 1 ngày làm việc thật ở mức hiện tại) | Người quyết định |
|---|---|---|
| 5 → 10 | Không `RATE_LIMIT` kéo dài hơn một cửa sổ reset; free RAM luôn ≥ 4 GB lúc đỉnh; tỷ lệ attempt FAILED không tăng so với mức 5; không `OWNERSHIP_VIOLATION` do chồng workspace | Owner (thay đổi Settings, có audit) |
| 10 → 18 | Như trên + RAM ≥ 32 GB hoặc benchmark task nặng đạt; quota còn ≥ 20% cuối cửa sổ; đã xác minh điều khoản nhiều phiên song song | Owner |

Hệ thống chỉ **hiển thị khuyến nghị**, không tự nâng. Khi gặp `RATE_LIMIT` hoặc host guard, tự **giảm** `effective_concurrency` tạm thời (FR-245) và phát `RESOURCE_CONCURRENCY_REDUCED`.

## 5. WBS và xử lý phụ thuộc

### 5.1 Từ yêu cầu đến task [Đề xuất]

1. Stage Requirement Intake: PM agent đọc prompt, trả JSON `proposedTasks[]` theo schema cố định: `{tempId, title, description, requiredRole, requiredSkills[], complexity 1..5, priority, dependsOn[tempId], writePatterns[], readPatterns[], acceptanceCriteria[], estimatedMinutes?}`.
2. `TaskService.validateWbs()` (mã xác định, không LLM) kiểm: không có chu trình; mỗi `dependsOn` tồn tại; mỗi task có ≥ 1 acceptance criterion kiểm chứng được; `writePatterns` hợp lệ (không `..`, nằm trong ownership của role); không hai task **chạy song song** có `writePatterns` giao nhau (nếu giao, tự thêm cạnh tuần tự và báo); số task ≤ ngưỡng (mặc định 60/lần) để tránh spawn tax.
3. Hợp lệ → tạo `tasks` ở `BACKLOG` và `task_dependencies` (`finish_to_start`); owner/policy duyệt (xem §15 câu 2: có auto-approve complexity ≤ 2 hay không).
4. WBS lồng nhau: `tasks.parent_task_id`; task cha chỉ `COMPLETED` khi mọi con `COMPLETED` **và** evidence cấp cha đủ.

### 5.2 Từ WBS đến node_run

WBS có thể sinh ra **node động** trong run bằng node kind `task` (xem `WORKFLOW_ENGINE.md` §4.2): graph JSON không đổi, chỉ thêm hàng `node_runs` với `nodeKey='dyn:<ulid>'` và nối cạnh theo `task_dependencies`. Scheduler coi node động như node thường (A07 §5.4).

### 5.3 Ready-set và trạng thái BLOCKED/READY

Task `READY` khi mọi dependency `COMPLETED`, project không pause, budget còn, và (nếu có) mọi node_run upstream `SUCCEEDED`. Scheduler **kiểm lại trong transaction** (event `TaskReady` chỉ là gợi ý tick sớm; tránh race, A13 §14). Task `BLOCKED` luôn có `blocked_reason` thuộc tập: `DEPENDENCY_FAILED`, `NO_ELIGIBLE_AGENT`, `PROVIDER_DOWN`, `OWNERSHIP_CONFLICT`, `BUDGET`, `OPERATOR`, `EMERGENCY_STOP`.

### 5.4 Ưu tiên và đường găng [Đề xuất]

Priority hiệu dụng gồm urgency (deadline), `criticalPathBoost` (task có ≥ 3 dependent trực tiếp +15), `ageBoost` (tối đa +20, chống starvation) và phạt công bằng giữa project (A13 §5.2, §9). Đường găng tính lại khi task hoàn thành hoặc thất bại; **không** dùng LLM.

### 5.5 Thay đổi dependency khi đang chạy [Đề xuất]

Output upstream bị sửa sau khi task con đã dùng: phát `DEPENDENCY_NOTIFICATION(CHANGED)` (`AGENT_COLLABORATION.md` §4). Con đang chạy nhận cảnh báo ở turn kế; con đã `COMPLETED` thì PM đề xuất task "re-validate", Orchestrator quyết định mở lại. Đây là cơ chế duy nhất cho phép "mở lại" task đã đóng, và luôn tạo **bản revision mới**, không sửa lịch sử.

## 6. Giao việc: AUTO và MANUAL

| | AUTO (mặc định) | MANUAL |
|---|---|---|
| Ai quyết | `AssignmentService.pick()` | Owner chọn agent (FR-110) |
| Cơ chế | Hard filter (role, skill bắt buộc, permission, ownership, provider khả dụng, budget, workload, workspace) rồi chấm điểm mềm (role/skills/tier/continuity/load/time/phạt lỗi gần đây); tie-break: ít session gần nhất, chi phí ước tính thấp, id ổn định | Hệ thống **vẫn kiểm hard filter** về permission, ownership, provider, budget; owner chỉ ghi đè được tiêu chí mềm và `requiredRole` |
| Ghi lại | `task_assignments.reason='workflow'` (DB) kèm `score breakdown` trong event `TASK_ASSIGNED` | `reason='manual'` + `assigned_by_user_id` |
| Không có ứng viên | Task ở `READY`, phát `ASSIGNMENT_BLOCKED`; PM đề xuất chọn/tạo role; sau 10 phút gửi notification cho owner | Từ chối thao tác kèm lý do hard filter bị vi phạm |

Công thức trọng số: A07 §6 (0.35 skill, 0.25 load, 0.15 provider free, 0.15 context, 0.10 health) và A13 §5.3 (role 0.25, skills 0.20, …) **khác nhau**. Đề xuất: dùng tập tiêu chí của A13 (đầy đủ hơn, có ownership/workspace), trọng số **cấu hình theo org** và coi các số là điểm khởi đầu cần hiệu chỉnh bằng dữ liệu thật [Giả định]. Mọi quyết định ghi `scheduler_event(kind='SCORE', breakdown)` để giải thích được.

Quy tắc bất biến: **một task có đúng một agent chịu trách nhiệm** (`ownerAgentId`); review/help từ agent khác không đổi owner. Đổi owner chỉ qua (a) reallocation, (b) Owner/PM đề xuất rồi Orchestrator thực hiện, đều ghi `task_assignments` và `TASK_REASSIGNED`/`TASK_HANDOVER`.

## 7. Tái phân bổ khi thất bại

### 7.1 Phân loại và hành động [Đề xuất, hợp nhất A07 §4.3, A13 §7-8, FR-111]

| failureClass | Tự retry? | Hành động mặc định | Đổi agent? |
|---|---|---|---|
| `TIMEOUT`, `CRASH`, `STALE`, `TOOL_ERROR` (idempotent) | Có, backoff mũ + jitter | Attempt mới (session + worktree mới), phạt agent vừa lỗi | Ưu tiên agent khác cùng role nếu có; nếu chỉ có một thì giữ sau backoff |
| `RATE_LIMIT` | Có, **không tiêu attempt** | Đóng băng cả provider pool tới reset/backoff; task `READY` với `queuedReason` | Không; không auto-switch provider |
| `AUTH` | Không | Agent `blocked`, task `BLOCKED(PROVIDER_DOWN)`, notify | Không tự động |
| `BUDGET_EXCEEDED` | Không | Hard stop; chờ owner nâng budget | Không |
| `EVIDENCE_MISSING` | Có (tối đa 1 lần với prompt nhắc rõ acceptance criteria) | Attempt mới với feedback "thiếu bằng chứng loại X" | Sau lần 2: handover |
| `OWNERSHIP_VIOLATION` | Có 1 lần | Diff gate loại file ngoài manifest, attempt mới | Sau lần 2: PM đề xuất chia lại task |
| `USER_CANCEL` | Không | Dừng, giải phóng | — |

Ghi chú: backoff `base 30 s, cap 15 phút, jitter ±20%`, `RATE_LIMIT` nhân 4 (A13 §7) và thống nhất với A07 (`30_000 * 2^(n-1)`, cap 15 phút).

### 7.2 Handover (FR-111) [Đề xuất]

Sau khi **hết retry** trên agent hiện tại (mặc định `tasks.max_retry`), task được chuyển cho agent khác đủ điều kiện, kèm: (1) `TaskBrief` (task, files + lý do, decisions, outputs upstream, blockers); (2) tóm tắt các attempt trước (lỗi, `failureClass`, đường dẫn log); (3) link branch/stash của attempt cuối (`reallocation/<taskId>/<attempt>`, A13 §8) để agent mới **tự quyết** dùng hay bỏ; (4) **không** chuyển transcript. Hết cả vòng handover (mặc định 1 lần) → task `FAILED` vĩnh viễn, notify owner, run đi theo nhánh `onBranchFailure` của node.

### 7.3 Quy trình khi session `LOST`/`FAILED` có thể retry

1. Nhả semaphore/lease; đánh dấu workspace "dirty".
2. `git status` trong worktree; có thay đổi thì lưu vào nhánh/stash tham chiếu.
3. `task.retry_count++`; nếu vượt `max_retry` → handover (7.2).
4. Ngược lại task `READY` với `scheduledNotBefore = now + backoff`; scoring phạt agent vừa lỗi.
5. Phát `TASK_FAILED`(attempt) hoặc `WORKFLOW_NODE_RETRY_SCHEDULED`, và `AGENT_STATE_CHANGED`.

## 8. Quy tắc hoàn thành dựa trên bằng chứng: "không COMPLETED nếu thiếu acceptance evidence"

### 8.1 Phát biểu quy tắc [Đề xuất] — [Cần người dùng phê duyệt]

> Một task **chỉ** được chuyển sang `COMPLETED` (và node_run sang `SUCCEEDED`, agent sang `completed`) khi `EvidenceService` đã ghi **ít nhất một bằng chứng nghiệm thu hợp lệ cho từng acceptance criterion** của task. Agent tự báo "xong", exit code 0 của session, hay tóm tắt của agent **không phải** bằng chứng.

Quy tắc này hiện thực hóa gate "COMPLETED cần bằng chứng" nêu trong `WORK_PLAN.md`/`AGENT_ORGANIZATION.md` §9 và tách biệt người viết với người nghiệm thu.

### 8.2 Loại bằng chứng hợp lệ theo loại node/task

| Loại | Bằng chứng tối thiểu | Ai/cái gì tạo |
|---|---|---|
| Code (Agent/Task dev) | (a) Diff trong ownership manifest do GitService xác minh; (b) kết quả test liên quan `exit 0` (artifact `test_result`) khi task có test; (c) review verdict `PASS` từ agent **khác** tác giả | GitService, Test node, Reviewer agent |
| Tài liệu (BA, Architect, UI/UX) | File sinh ra đúng đường dẫn, qua lint/validate cấu trúc, và approval gate (nếu stage có gate) | GitService, ApprovalService |
| Test node | `exit_code`, báo cáo (artifact `test_result`) | Test worker |
| Deployment preview | Build thành công + `PREVIEW_READY` + health check URL | Deploy worker |
| Approval node | Quyết định của người (`approvals.decision`) | ApprovalService |
| Bug/Fix task | Test tái hiện lỗi chuyển từ fail sang pass | Test node |

### 8.3 Ràng buộc độc lập

1. Tác giả task **không** được xác nhận evidence của chính mình (agent ≠ reviewer; reviewer thuộc role khác, thường `reviewer` hoặc `qa`).
2. Evidence phải **gắn với phiên bản cụ thể** (commit sha/artifact sha256 tại thời điểm xác minh); sửa code sau đó làm evidence **hết hiệu lực** (cần chạy lại).
3. Test do chính agent sửa để "xanh giả" bị chặn: diff gate đánh dấu thay đổi vào file test của task không thuộc `writePatterns`; template giao việc ghi rõ "không sửa test để pass giả" (A07 §9).

### 8.4 Điểm cưỡng chế

| Lớp | Cơ chế |
|---|---|
| Ứng dụng | `TaskService.complete()` là **hàm duy nhất** set `COMPLETED`; nó gọi `EvidenceService.assertSatisfied(taskId)` trong transaction |
| Cơ sở dữ liệu | [Cần duyệt] trigger trên `tasks`: `status='COMPLETED'` yêu cầu tồn tại bản ghi evidence hợp lệ cho từng criterion; owner có thể chỉ dùng lớp ứng dụng ở MVP |
| Workflow | node_run chỉ `RUNNING → SUCCEEDED` khi guard evidence đạt (xem `WORKFLOW_ENGINE.md` §5) |
| Kiểm toán | `TASK_EVIDENCE_RECORDED` và `TASK_COMPLETED` mang `evidenceIds`; UI hiển thị bằng chứng cạnh trạng thái |

### 8.5 Lưu trữ (khoảng trống schema) [Đề xuất]

`DATABASE_DESIGN.md` có `tasks.acceptance_criteria[]`, `tasks.result.artifactIds`, bảng `artifacts` nhưng **chưa có** liên kết "criterion ↔ evidence". Đề xuất bảng mới `task_evidence(id, task_id, criterion_index, kind, artifact_id, ref_sha, verified_by_kind, verified_by_id, verdict, verified_at, superseded_by)`. Nếu owner không muốn thêm bảng, thay bằng JSONB trong `tasks.result.evidence[]` (mất khả năng ràng buộc DB).

### 8.6 Khi thiếu evidence

Attempt `FAILED(EVIDENCE_MISSING)`; hệ thống sinh feedback chỉ rõ criterion nào thiếu loại bằng chứng gì; retry theo §7.1; sau đó handover; cuối cùng `BLOCKED` chờ owner (owner **có thể** ký chấp nhận thủ công, ghi nhận như evidence loại `HUMAN_WAIVER` có audit, không mặc định).

## 9. Vòng revision khi người dùng reject

### 9.1 Nguyên tắc [Đề xuất trên FR-128, FR-129, FR-135, A07 §8]

Không chạy lại toàn bộ workflow. Chỉ tạo **fix task có mục tiêu** cho phần bị ảnh hưởng; mỗi vòng tăng số phiên bản deliverable (FR-129).

### 9.2 Các bước

| # | Bước | Chủ thể |
|---|---|---|
| 1 | `HUMAN_REJECTED` nhận `rejectItems[]` (file/khu vực, severity, text, markup ảnh nếu có, FR-130) | ApprovalService |
| 2 | Tạo checkpoint `before_revision`; `run.revision += 1` | Scheduler |
| 3 | Spawn node động `dyn:pm-triage:<ulid>` cho PM agent: input `rejectItems` + các node agent upstream trong cùng stage + `summary` của chúng; output `fixTasks[] {targetNodeId, assignee: same|auto, brief, scope:'patch'}` | PM agent (đề xuất) |
| 4 | Validate `fixTasks` (target phải nằm upstream của gate bị reject; brief có acceptance criteria). PM trả JSON sai 2 lần: fallback tạo fix task cho mọi agent node upstream trực tiếp của gate | RevisionService |
| 5 | Tạo node_run bản revision mới cho target và mọi node trên đường target → gate (merge, tool, test, review); node khác giữ nguyên | Scheduler |
| 6 | Fix task chạy trên branch cắt từ branch của attempt `SUCCEEDED` trước (patch, không làm lại từ `main`); artifact cho reviewer kèm **diff so với revision trước** | GitService |
| 7 | Quay lại gate (FR-135); reviewer thấy lịch sử phiên bản | ApprovalService |
| 8 | Vượt `maxRevisions` → run `FAILED(REVISION_LIMIT)` + notification | Scheduler |

Gate mặc định bắt buộc theo FR-199: Business Analysis (stage 2), UI/UX (stage 4), Human Acceptance (stage 8). Reject ở stage 8 mới cần triage nhiều stage (lỗi code vs thiết kế); PM đề xuất `targetNodeId` có thể ở stage 5 hoặc 4.

### 9.3 Đánh số phiên bản [Đề xuất]

`run.revision` toàn cục (R0, R1, …); `node_runs` có bản ghi theo `(run, nodeKey, revision/attempt)`; deliverable (PRD, thiết kế, build) có `version` tăng theo mỗi lần được approve hoặc reject; UI hiển thị `Stage 8 · R2`. **Lưu ý schema**: DB `node_runs` unique `(workflow_run_id, node_key, attempt)`, A07 dùng `(runId, nodeId, revision)` và `attempt` riêng. Cần cột `revision` bổ sung hoặc mã hóa vào `attempt` (xem M-O4).

## 10. Hồi phục sau restart

### 10.1 Nguyên tắc [Đề xuất, A07 §2.1, A13 §11]

Postgres là nguồn sự thật; Redis (semaphore, heartbeat TTL, stream buffer) **xóa trắng được bất cứ lúc nào**; BullMQ chỉ là transport. Chuyển trạng thái = transaction + outbox. Tick là hàm idempotent (`jobId = sched:<runId>`).

### 10.2 Trình tự khởi động của `RecoveryService`

1. **Khóa**: xin advisory lock "recovery" (một replica làm); tick loop và workers **chưa bật**.
2. **Quét session** `starting|active|suspended|resumable` kèm `pid`: kiểm process còn sống bằng `tasklist`/`Get-Process` **và** command line/env chứa `AIW_EXECUTION_ID` (chống PID reuse).
   - Chết → session `crashed`, task `FAILED(CRASH)` → §7.
   - Còn sống nhưng pipe stdout đã mất: **mặc định kill tree + reallocate** (adopt mù mất evidence; A13 §11). Adopt chỉ khi provider hỗ trợ attach (`capabilities().attach`), [Giả định].
3. **Node/task mồ côi**: node_run `QUEUED` không có job BullMQ → `READY`; `RUNNING` không heartbeat > 3 phút → `FAILED(STALE)`; approval `pending` còn hạn → giữ nguyên (không mất, vì không giữ worker); approval quá `expires_at` → `expired` theo `onTimeout`.
4. **Rebuild Redis**: `sem:global`, `sem:provider:*`, `sem:project:*` = COUNT từ session đang giữ slot thật; presence/office positions từ `character_positions` (write-behind) và từ replay event gần nhất; stream buffer rỗng cũng được (client sẽ đọc DB).
5. **Outbox**: relay đọc lại mọi event `published=false` theo `global_seq`, phát **at-least-once**; consumer idempotent theo `event.id` (`EVENT_SYSTEM.md` §7).
6. **Run**: mọi run `RUNNING|WAITING_HUMAN` → enqueue `tick` (idempotent). Run `PAUSED` giữ nguyên.
7. **Worktree**: với session crashed, lưu thay đổi dở vào nhánh/stash tham chiếu rồi nhả workspace lock.
8. Mở tick loop và workers; phát `SYSTEM_RECOVERED` kèm số liệu (đã adopt, đã kill, đã reallocate, approval giữ nguyên).

### 10.3 Kịch bản và kết quả mong đợi

| Kịch bản | Kết quả |
|---|---|
| Kill -9 API khi 3 session đang chạy | Ghi nhận mỗi session theo 10.2(2); không spawn đôi; không mất approval; run tiếp tục sau tick |
| Redis flush giữa chừng | Semaphore rebuild từ Postgres; không vượt limit; không mất event đã commit |
| Postgres restart ngắn | Worker retry kết nối; không có transition nào bị ghi nửa chừng nhờ transaction |
| Reboot Windows | Mọi session `crashed` (process không còn); task retry theo backoff; approval giữ nguyên |
| Hai replica scheduler cùng lúc | `SELECT ... FOR UPDATE` trên run + `jobId` dedupe, chỉ một tick hiệu lực |

Mốc thời gian (heartbeat 30 s, lease 60 s gia hạn 20 s, stale 3 phút, `spawn_timeout` 60 s) lấy từ A07 §4.4 và A13 §10; là [Đề xuất] chưa kiểm thử thực.

## 11. Đã xác minh và Đề xuất

| Hạng mục | Trạng thái | Căn cứ |
|---|---|---|
| Đo RAM/CPU/latency 1 và 3 process `claude -p`; không rate-limit ở 3 process | **Đã xác minh** (thực nghiệm 2026-10-30) | `RESOURCE_MANAGEMENT.md` §3 |
| Tài khoản Claude là claude.ai `team`, một credential dùng chung | **Đã xác minh** | `claude auth status` |
| Docker daemon chưa chạy; Codex CLI chưa cài | **Đã xác minh** (lúc kiểm tra) | `DECISION_LOG.md` |
| ChatDev 2.0 chạy in-memory, không checkpoint; OpenHands event là append-only có discriminator | **Đã xác minh bằng đọc mã nguồn** | `CHATDEV_ANALYSIS.md`, `OPENHANDS_ANALYSIS.md` |
| LangGraph `interrupt()` chạy lại node, Temporal tốn vận hành, BullMQ Flows là tree | **Đã xác minh bằng tài liệu chính thức** (không chạy PoC) | `ORCHESTRATION_WORKFLOW_RESEARCH.md` |
| Conductor (DAG executor trên BullMQ + Postgres) chạy đúng với kill -9, Redis flush, 2 replica | **Đề xuất** — chưa có PoC | A07 §10 liệt kê test nhưng chưa chạy |
| Scoring, trọng số, đường găng, ngưỡng heartbeat/stale | **Đề xuất** | A07/A13 |
| Claude `--resume` với `-p`, pause/resume an toàn, attach lại pipe | **Giả định** | A13 §7, §11 |
| Hành vi Codex (`app-server`, rate-limit, resume) | **Giả định** | Codex chưa cài |
| Quota 5 giờ/tuần của subscription, điều khoản nhiều phiên song song | **Giả định / Cần người dùng phê duyệt** | `RESOURCE_MANAGEMENT.md` §8 |
| 5/10/18 agent trên máy này | **Giả định** (chỉ 1 và 3 được đo) | §4.3 |
| Quy tắc evidence, bảng `task_evidence`, Chief Orchestrator | **Đề xuất** — [Cần người dùng phê duyệt] | Tài liệu này |

## 12. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| MO-01 | Chief Orchestrator = mã xác định + Owner; PM agent chỉ đề xuất | LLM không giữ quyền hệ thống; hệ thống chạy khi PM lỗi | [Đề xuất] — [Cần người dùng phê duyệt] |
| MO-02 | Assignment xác định, test được; PM chỉ gợi ý/phá hòa | Giải thích được từng quyết định; không tốn token | [Đề xuất] |
| MO-03 | Default 5, max 18, chỉ nâng khi có bằng chứng | D-010, đo thực tế chỉ tới 3 | [Đề xuất] trên số đo [Đã xác minh] |
| MO-04 | Hết slot: task chờ ở `READY` trong Postgres | Không stalled, không giữ lock BullMQ | [Đề xuất] (A07) |
| MO-05 | Không auto-switch provider khi hết quota | Tiền và nhất quán là quyết định owner | [Đề xuất] |
| MO-06 | "Không COMPLETED nếu thiếu evidence", người nghiệm thu độc lập tác giả | Yêu cầu owner; chặn tự báo xong | [Đề xuất] — [Cần người dùng phê duyệt] |
| MO-07 | Revision tạo fix task có mục tiêu, không chạy lại workflow | FR-128 | [Đã xác minh] yêu cầu, [Đề xuất] cách làm |
| MO-08 | Sau crash: kill tree + reallocate, không adopt mù | Không attach lại được pipe; giữ evidence | [Đề xuất] |

## 13. Dependencies

`AGENT_ARCHITECTURE.md` (state, session), `WORKFLOW_ENGINE.md` (node kinds, tick, approval), `EVENT_SYSTEM.md` (outbox, replay), `DATABASE_DESIGN.md` (thêm `task_evidence`, cột revision, enum node_run), `RESOURCE_MANAGEMENT.md` (semaphore, budget), `WORKSPACE_AND_GIT_ISOLATION.md` (worktree, manifest, diff gate), `SECURITY.md` (hook, Docker), A05/A06 (resume, usage, rate-limit thực tế).

## 14. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Quota dùng chung cạn trước khi 18 agent có ích | Mặc định 5; chỉ nâng theo §4.4; hiển thị reset time; không auto-switch |
| Evidence gate làm chậm hoặc kẹt task hợp lệ | Feedback cụ thể, retry có mục tiêu, `HUMAN_WAIVER` có audit |
| PM trả WBS xấu (quá nhiều task nhỏ, chu trình) | Validator xác định, ngưỡng số task, gộp task complexity 1 cùng role |
| Triage sai gây sửa không đúng chỗ | Validate target upstream của gate; fallback sửa mọi upstream trực tiếp; owner thấy diff |
| Conductor chưa có PoC | Phase 1 làm spike recovery (kill -9, Redis flush, 2 replica) trước khi xây tiếp |
| PID reuse, kill không sạch cây trên Windows | `AIW_EXECUTION_ID`, `taskkill /T /F`, test emergency stop |
| Hai nguồn sự thật cho state (agent/task/node/session) | Một service transition, test property ở `AGENT_ARCHITECTURE.md` |

## 15. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Thuật ngữ và phạm vi Chief Orchestrator (§3.1); có cần Chief Orchestrator Agent LLM không.
2. [Cần người dùng phê duyệt] Task do PM đề xuất có auto-approve cho complexity ≤ 2 hay mọi task cần duyệt (`AGENT_COLLABORATION.md` §14.1).
3. [Cần người dùng phê duyệt] Cho phép `HUMAN_WAIVER` thay evidence không, và ai được ký.
4. [Cần người dùng phê duyệt] Giới hạn revision: A07 mặc định 3, FR không nêu giới hạn.
5. Điều khoản dịch vụ nhiều phiên song song; gói quota thật (`RESOURCE_MANAGEMENT.md` §15.1).
6. Merge conflict ở stage Development (OI-FR-08): đã có nhánh condition `conflict` trong template, cần owner xác nhận người giải quyết (agent hay người).
7. Preview chạy ở đâu (OI-FR-09), ảnh hưởng điều kiện evidence của deployment preview.

### Mâu thuẫn nguồn liên quan

| ID | Mâu thuẫn | Xử lý |
|---|---|---|
| M-O1 | FR-109: "PM agent chấm điểm"; A07 §6/A13: scoring xác định, PM chỉ can thiệp khi bế tắc | Theo A07/A13 (§3.4) |
| M-O2 | Global cap: A07 = 10 và per-provider 6; D-010/FR-231 = 5 mặc định, max 18; DB `provider_configs.max_concurrent` = 2; A13 claude 5, codex 3 | D-010 và FR thắng cho global; cap provider do `ResourceConfig`, giá trị theo D-010 APPROVED: Claude 5, Codex 3; DB default đã đổi 2 thành 5 |
| M-O3 | A07 dùng `node_attempts` + `attempt` trong BullMQ, DB `node_runs.attempt` + `agent_sessions`; A07 retry trong node, A13 retry ở task | Một attempt = một `agent_sessions`; `node_runs.attempt` đếm; `tasks.retry_count` phản chiếu |
| M-O4 | A07: `NodeRun` unique `(runId, nodeId, revision)`; DB: unique `(run, node_key, attempt)`, không có `revision` | Thêm cột `revision` (migration) |
| M-O5 | A07/A13 dùng Prisma; D-002 chọn Drizzle | Drizzle thắng (D-002 APPROVED, 2026-10-30); mã giả Prisma trong A07 chỉ là minh họa |
| M-O6 | Worktree: A07 `wt/<run>/<node>/<attempt>` và `wf/<runId>/<nodeId>/r<rev>`; D-009 `agent/<slug>/<task-id>` | D-009 thắng |
| M-O7 | A07 đếm `agent_status` kiểu `working`; A13 TaskState có `DONE/DRAFT/PAUSED/QUEUED`; DB `task_status` 10 giá trị với `COMPLETED` | Dùng 10 giá trị DB; `QUEUED` chỉ là `queuedReason` |
| M-O8 | `task_status` UPPERCASE, enum khác lowercase (DECISION_LOG mâu thuẫn 5) | Tài liệu viết HOA; casing lưu trữ chờ owner |

## 16. Acceptance Criteria (Phase 1)

- [ ] Test đồng thời: `max_concurrent_agents=2`, 5 task `READY` → đúng 2 session `active`, 3 task `READY` với `queuedReason`; đổi lên 4 có hiệu lực ở tick kế tiếp; giảm không giết session đang chạy.
- [ ] 50 acquire đồng thời với limit 5 → đúng 5 thành công (Lua), Redis flush giữa chừng không vượt limit.
- [ ] `validateWbs` từ chối chu trình, dependency không tồn tại, hai task song song giao `writePatterns`, task không có acceptance criterion.
- [ ] AUTO: mỗi quyết định có `scheduler_event` với breakdown; MANUAL vẫn bị chặn nếu vi phạm permission/ownership.
- [ ] Sau `max_retry` thất bại, task handover cho agent khác kèm TaskBrief và tóm tắt attempt, không kèm transcript; hết vòng handover thì `FAILED` + notification.
- [ ] Không có đường nào đặt `COMPLETED` ngoài `TaskService.complete()`; test: task thiếu evidence cho một criterion không thể hoàn thành; tác giả không tự xác nhận evidence của mình; sửa code sau xác minh làm evidence hết hiệu lực.
- [ ] Reject ở gate tạo checkpoint, `run.revision+1`, fix task có mục tiêu; node ngoài đường ảnh hưởng giữ nguyên; PM trả JSON sai 2 lần thì fallback chạy được; vượt `maxRevisions` thì `FAILED(REVISION_LIMIT)`.
- [ ] Kill -9 API giữa lúc 3 session chạy: không spawn đôi, không mất approval, run tiếp tục; Redis flush và Postgres restart đều qua test; hai replica scheduler không tick trùng.
- [ ] Emergency stop: sau 15 s không còn process con nào; mọi task RUNNING sang `BLOCKED(EMERGENCY_STOP)`.
- [ ] Owner đã duyệt các mục [Cần người dùng phê duyệt] ở §15 trước khi Phase 1 bắt đầu.
