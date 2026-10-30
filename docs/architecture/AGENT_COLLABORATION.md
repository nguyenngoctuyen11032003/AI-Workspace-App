# AGENT_COLLABORATION.md — Message bus, context chính thức và cộng tác giữa agent

Tác giả: Agent 13 — Phase 0, chỉ tài liệu. Ngày: 2026-10-30. Trạng thái: DRAFT.
Liên quan: `AGENT_ORGANIZATION.md` (role, ownership), `AGENT_SCHEDULER.md` (ownership task, reassign), `RESOURCE_MANAGEMENT.md` (token budget). Workflow engine và cơ chế chạy DAG: xem `ORCHESTRATION_WORKFLOW_RESEARCH.md` (A07).

Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].

---

## 1. Mục đích

Định nghĩa cách 18 agent (mỗi agent là một CLI subprocess độc lập, không chia sẻ bộ nhớ) trao đổi với nhau: ngữ nghĩa message bus, quy tắc tiết kiệm token khi truyền ngữ cảnh, ba tầng context chính thức (global / agent / task), mẫu "họp" (meeting), xử lý xung đột, lưu trữ bền, và ranh giới giữa **PM agent** (một agent như mọi agent khác) với **Orchestrator service** (code NestJS).

## 2. Phạm vi

- Trong: message types & semantics, delivery model, token economy, context management, meeting pattern, conflict handling, persistence, PM vs orchestrator.
- Ngoài: DAG/workflow state (A07), UI chat/office (A11), transport chi tiết WS gateway (Phase 1), cơ chế inject prompt vào CLI (A05/A06).

## 3. Mô hình cơ bản [Đề xuất]

Agent CLI **không** nhận message realtime giữa chừng một turn (stdin của `claude -p` là prompt một lần; [Giả định] stream-json input có thể cho phép multi-turn — A05 xác minh). Do đó message bus là **mailbox-based, asynchronous**:

- Mỗi `AgentInstance` có một **inbox** trong Postgres (`agent_message`).
- Message được **giao** (deliver) cho agent ở hai thời điểm: (1) khi spawn execution mới (gộp vào prompt), (2) khi execution đang chạy kết thúc một turn và orchestrator quyết định mở turn tiếp (resume session với message mới).
- Orchestrator service là **người đưa thư duy nhất**; agent không gọi nhau trực tiếp. Agent "gửi" message bằng cách ghi **structured output** (JSON block trong kết quả, hoặc gọi MCP tool `aiw.send_message` do app cung cấp — [Đề xuất], A05/A12 hiện thực).

Hệ quả: độ trễ message = thời gian tới turn kế tiếp của người nhận (giây tới phút). Chấp nhận được cho cộng tác kiểu "ticket", không phù hợp "chat realtime" — chat realtime là giữa **user và agent**, không phải agent–agent.

## 4. Message types và ngữ nghĩa [Đề xuất]

Mọi message có header chung:

```ts
interface AgentMessage {
  id: string; type: MessageType; projectId: string;
  from: { kind: 'agent'|'orchestrator'|'user'; id: string };
  to: { kind: 'agent'|'role'|'broadcast'; id?: string };    // role: gửi cho instance ACTIVE của role đó
  taskId?: string; threadId?: string; replyTo?: string;
  priority: 'low'|'normal'|'high'|'urgent';
  payload: Record<string, unknown>;        // theo type, validate bằng zod
  artifacts?: ArtifactRef[];               // {kind:'file'|'diff'|'doc'|'log', path, sha256, bytes}
  createdAt: Date; deliveredAt?: Date; acknowledgedAt?: Date; expiresAt?: Date;
}
```

| Type | Ai gửi → ai nhận | Payload chính | Ngữ nghĩa / hiệu ứng phụ |
|---|---|---|---|
| `AGENT_MESSAGE` | agent → agent/role | `text`, `context: 'fyi'|'question'` | Thông tin hoặc câu hỏi; không đổi trạng thái task. Câu hỏi có `expiresAt`; quá hạn → gửi lại cho user |
| `TASK_HANDOFF` | agent (owner) → agent/role | `summary`, `doneSoFar[]`, `remaining[]`, `files[]`, `decisions[]`, `blockers[]` | Đề nghị chuyển ownership. **Orchestrator** mới là người đổi `ownerAgentId` (qua scheduler reassign), sau khi người nhận `ACCEPT` hoặc PM/user duyệt. Người gửi vẫn là owner tới lúc đó |
| `REQUEST_REVIEW` | agent → role `reviewer` (hoặc role chỉ định) | `diffRef`, `scope`, `checklist[]`, `acceptanceCriteria[]` | Tạo task con kiểu REVIEW gắn `parentTaskId`; task cha chuyển `REVIEW`; kết quả `REVIEW_RESULT{verdict: PASS/CHANGES_REQUESTED/FAIL, findings[]}` |
| `REQUEST_HELP` | agent → role/agent | `problem`, `triedSoFar[]`, `files[]`, `question` | Người nhận trả lời bằng `AGENT_MESSAGE(replyTo)`; không đổi owner. Nếu người nhận cần sửa file ngoài ownership → phải qua `TASK_HANDOFF` hoặc tạo task mới |
| `DEPENDENCY_NOTIFICATION` | orchestrator → agent | `dependencyTaskId`, `status: DONE/FAILED/CHANGED`, `outputs[]` | Tự động khi task phụ thuộc đổi trạng thái; nếu `CHANGED` (output cũ bị sửa) agent đang chạy nhận cảnh báo "input stale" ở turn kế |
| `CONTEXT_UPDATE` | orchestrator → broadcast/role | `scope: 'global'|'project'`, `docRef`, `version`, `changeSummary` | PRD/ADR/standards đổi version; agent nên đọc lại phần liên quan |
| `ARTIFACT_SHARED` | agent → agent/role/broadcast | `artifacts[]`, `note` | Chia sẻ file/diff/log; **tham chiếu**, không nhúng nội dung (xem §5) |
| `MEETING_INVITE` / `MEETING_INPUT` / `MEETING_MINUTES` | orchestrator/PM → role list | xem §7 | Họp bất đồng bộ |
| `APPROVAL_REQUEST` | agent → user (qua orchestrator) | `action`, `risk`, `command?` | L4 hoặc chuyển provider [Q-08/OWR-2: không tự động chuyển provider khi hết quota; thông báo Owner và chờ quyết định; Q-05: chờ tối đa 540 s rồi `WAITING_HUMAN`]; agent PAUSED/WAITING_APPROVAL |
| `ACK` | agent → orchestrator | `messageId`, `status: 'read'|'accepted'|'declined'`, `reason?` | Dùng cho HANDOFF/REVIEW/MEETING |

Ownership bất biến: **không message nào tự đổi owner của task**; chỉ scheduler (orchestrator) đổi và ghi `scheduler_event`.

## 5. Token economy — truyền gì, không truyền gì [Đề xuất]

Nguyên tắc: *agent nhận đúng những gì cần để làm task, dưới dạng tham chiếu có thể mở, không phải bản sao.*

**Được truyền** (theo thứ tự ưu tiên khi thiếu chỗ):
1. Task description + acceptance criteria (bắt buộc).
2. Danh sách file liên quan: đường dẫn + 1 dòng lý do (agent tự đọc bằng tool; không nhúng nội dung).
3. Decisions áp dụng: ADR id + tiêu đề + 1 câu tóm tắt (link tới file).
4. Dependencies: task cha id, trạng thái, **outputs** (đường dẫn + tóm tắt ≤ 300 ký tự mỗi output).
5. Blockers/known issues liên quan.
6. Message inbox chưa đọc liên quan task (tối đa N=10, cũ hơn → "còn K message, dùng tool để lấy").

**Không truyền**: toàn văn PRD/SRS (chỉ section liên quan hoặc link), log đầy đủ của task khác, lịch sử chat của agent khác, diff lớn (gửi `diffRef`, agent dùng `git diff`), nội dung đã có trong workspace (agent có file system).

Ngân sách context khi spawn: `context_budget_tokens` (default 8 000 cho phần orchestrator inject, không tính system prompt của CLI). Orchestrator cắt theo thứ tự ưu tiên ngược (6 → 2) và ghi `truncated: true` vào task context để agent biết hỏi thêm. Lý do con số: thực nghiệm đo prompt cố định của CLI đã ~48k token/spawn [Đã xác minh]; thêm 8k là ~15% — chấp nhận được; nếu tối giản config (RESOURCE_MANAGEMENT §4.3) có thể nâng lên 16k.

Đo lường: mỗi spawn ghi `injectedTokens` (ước tính bằng tokenizer gần đúng) vào ledger để tối ưu sau.

## 6. Quản lý context chính thức [Đề xuất]

| Tầng | Nội dung | Nguồn sự thật | Ai được sửa | Cách agent nhận |
|---|---|---|---|---|
| **Global/Project context** | PRD, SRS, SYSTEM_ARCHITECTURE, coding standards, ADR, glossary, DECISIONS log | File trong repo (`docs/**`) + bảng `context_document(version, sha)` | Role chủ sở hữu (ba/architect/…) qua task; mọi thay đổi tăng version và phát `CONTEXT_UPDATE` | Phần "Project brief" ≤ 1 500 token được orchestrator sinh tự động từ metadata + link; agent đọc file khi cần |
| **Agent context** | Template prompt (role), overrides, skill list, "memory" của agent: bài học, convention đã học, danh sách task đã làm (id + 1 dòng) | `AgentInstance` + `agent_memory` (append-only, có tóm tắt định kỳ) | Agent tự đề xuất ghi nhớ qua structured output `remember[]`; orchestrator lưu; user có thể xóa | Inject tối đa 1 000 token "memory digest" khi spawn |
| **Task context** | Mô tả, AC, files, decisions, dependency outputs, inbox liên quan, attempt history (tóm tắt lần fail trước) | `task` + `task_context_snapshot` (freeze tại thời điểm spawn để tái lập) | Orchestrator tổng hợp; PM/user sửa task | Toàn bộ, trong `context_budget_tokens` |

Snapshot: mỗi execution lưu đúng context đã inject (`task_context_snapshot`) → tái lập được "agent đã biết gì khi quyết định X" (phục vụ review/post-mortem và evidence cho owner).

Memory của agent: giới hạn 200 entry; khi vượt, orchestrator tạo task cho chính agent đó "tóm tắt memory" (hoặc role `docs`) — tránh prompt phình.

## 7. Meeting pattern (họp bất đồng bộ) [Đề xuất]

Dùng khi cần ý kiến nhiều role (ví dụ chốt API contract, triage lỗi). Không chạy 18 agent cùng lúc để "chat"; thay vào đó:

1. Người triệu tập (PM agent, hoặc user) tạo `Meeting{topic, question, participants[roles], deadline, maxRounds=2}`.
2. Orchestrator gửi `MEETING_INVITE` + context gọn (≤ 2 000 token) tới từng participant; mỗi participant trả lời bằng **một execution ngắn** (task ẩn `MEETING_INPUT`, complexity 1, budget nhỏ) — chạy tuần tự hoặc song song theo slot như task thường.
3. Hết round hoặc deadline, orchestrator gửi mọi input tới **một** agent tổng hợp (mặc định PM; với kỹ thuật: architect) → `MEETING_MINUTES{decisions[], actionItems[], dissent[]}`.
4. Minutes lưu vào `docs/decisions/MEETING-<id>.md` và phát `CONTEXT_UPDATE`. Action items → task mới (PM đề xuất, user/orchestrator tạo).
5. Nếu có `dissent` chưa giải quyết sau `maxRounds` → escalate user (`APPROVAL_REQUEST` kiểu `DECISION`).

Chi phí: N participants × 1 execution ngắn ≈ N × (spawn tax + ~1k token) — với spawn tax hiện tại 0.37 USD list, một cuộc họp 6 role ≈ 2.5 USD list; cần tối giản config để rẻ hơn.

## 8. Xử lý xung đột [Đề xuất]

| Loại xung đột | Phát hiện | Xử lý |
|---|---|---|
| Hai task muốn ghi cùng file | Scheduler hard-reject workspace (`AGENT_SCHEDULER.md` §5.3) | Chạy tuần tự; nếu cả hai cần gấp → PM/user chia nhỏ task |
| Agent ghi ngoài ownership | Hook PreToolUse (A09) chặn | Tool call fail, agent nhận thông báo → gửi `REQUEST_HELP`/`TASK_HANDOFF` cho owner đúng |
| Merge conflict giữa worktree | Khi orchestrator tích hợp branch | Tạo task `RESOLVE_CONFLICT` giao cho owner của task mới hơn; reviewer kiểm sau |
| Bất đồng kỹ thuật (review FAIL ↔ dev không đồng ý) | `REVIEW_RESULT` + `ACK(declined)` | Round 2 review với lý do; vẫn bất đồng → meeting (§7) hoặc escalate user. Reviewer **không** có quyền sửa code; dev **không** có quyền tự đánh dấu DONE |
| Dependency output đổi sau khi task con đã dùng | `DEPENDENCY_NOTIFICATION(CHANGED)` | Task con đang RUNNING nhận cảnh báo turn kế; đã DONE → PM quyết định mở lại (tạo task "re-validate") |
| Message lặp/vòng (A hỏi B, B hỏi A) | `threadId` depth > 6 hoặc > 3 round trong 10 phút | Orchestrator chặn, tạo meeting hoặc hỏi user |

## 9. PM agent làm gì — Orchestrator service làm gì [Đề xuất] — [Cần người dùng phê duyệt]

| Việc | PM agent (LLM, role 01) | Orchestrator service (code) |
|---|---|---|
| Phân rã yêu cầu thành task, ước lượng complexity/priority, đề xuất dependencies | **Đề xuất** (structured output `proposedTasks[]`) | Validate, tạo task sau khi user/auto-policy duyệt |
| Chọn agent cho task | Có thể gợi ý `preferredAgentId` | **Quyết định** qua scoring; có thể bỏ qua gợi ý |
| Đổi owner/reassign | Đề xuất | Thực hiện + audit |
| Theo dõi tiến độ, viết báo cáo, cập nhật WORK_PLAN/PROJECT_STATUS | **Làm** (như task thường, theo lịch) | Cung cấp dữ liệu (API/metrics) |
| Triệu tập họp, tổng hợp minutes | **Làm** | Chạy cơ chế meeting, lưu |
| Nhắc nhở agent trễ, đề xuất chia nhỏ task | **Làm** | Phát hiện stale/timeout, kill, retry (không cần PM) |
| Enforce limit/budget/permission | Không | **Làm** |
| Spawn/kill process, semaphore, recovery | Không | **Làm** |
| Giao tiếp với user về "vì sao chậm" | **Làm** (ngôn ngữ tự nhiên) | Cung cấp evidence (log, metrics) |

Nguyên tắc: **PM agent không có quyền hệ thống** — mọi thứ PM "làm" là tạo đề xuất/tài liệu; orchestrator là nơi duy nhất thực thi trạng thái. Điều này tránh LLM vô tình giết process hay vượt limit, và giữ hệ thống chạy được khi PM agent lỗi/hết quota.

## 10. Lưu trữ bền [Đề xuất]

- `agent_message` (append-only; index theo `to`, `taskId`, `threadId`, `deliveredAt IS NULL`), retention 90 ngày rồi archive sang file JSONL per project.
- `task_context_snapshot` (JSONB + sha), giữ vĩnh viễn trong vòng đời project (evidence).
- `agent_memory` (append-only + bảng digest).
- `meeting`, `meeting_input`, `meeting_minutes`.
- Artifact: file trong workspace/worktree + `artifact(kind, path, sha256, bytes, executionId)`; log stdout/stderr lưu file `logs/<executionId>.jsonl` (A08 quyết định có đưa vào DB không; đề xuất không, chỉ index).
- Mọi message/snapshot đều gắn `executionId` để truy vết "agent nói gì ở turn nào".

## 11. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Mailbox bất đồng bộ, orchestrator là người đưa thư duy nhất | CLI subprocess không nhận message giữa turn; kiểm soát token và audit tập trung | [Đề xuất] (giới hạn CLI: [Giả định] chờ A05) |
| Tham chiếu thay vì nhúng nội dung | Spawn tax đã ~48k token; agent có file system | [Đã xác minh] động cơ |
| Ownership không đổi qua message | Một agent chịu trách nhiệm (yêu cầu owner) | [Đề xuất] |
| Meeting = N execution ngắn + 1 tổng hợp, không chat realtime | Rẻ, có minutes, không phụ thuộc giữ 18 process mở | [Đề xuất] |
| PM agent chỉ đề xuất | An toàn, hệ thống vẫn chạy khi PM lỗi | [Đề xuất] — [Cần người dùng phê duyệt] |

## 12. Dependencies

A05 (stream-json input multi-turn, MCP tool do app cung cấp, hook PreToolUse), A06 (Codex tương đương), A07 (task cha/con, trạng thái REVIEW), A08 (bảng message/snapshot/memory), A09 (ownership hook), A11 (UI inbox/meeting/office view).

## 13. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Agent "nói chuyện" quá nhiều → tốn token, vòng lặp | Giới hạn thread depth, budget message per task (default 20), mọi message qua orchestrator |
| Structured output của agent sai format | Zod validate; sai → bỏ qua + ghi warning + nhắc trong turn kế; không crash |
| Context snapshot phình DB | JSONB nén; retention policy; chỉ lưu link cho file lớn |
| Memory agent học sai convention | User có thể xem/xóa memory; digest do reviewer kiểm định kỳ |
| PM agent đề xuất quá nhiều task nhỏ → spawn tax | Policy: task complexity 1 được gộp (batching) nếu cùng role và không phụ thuộc |

## 14. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Task do PM đề xuất có được tự động tạo (auto-approve) với complexity ≤ 2, hay mọi task cần user duyệt?
2. Cơ chế "agent gửi message": structured output trong kết quả vs MCP tool `aiw.send_message` — phụ thuộc A05 (MCP server nội bộ) và A12.
3. Agent có được đọc inbox của agent khác (minh bạch) không? Đề xuất: đọc được theo project, không đọc memory riêng.
4. Ngôn ngữ message: tiếng Việt hay tiếng Anh? (ảnh hưởng token; đề xuất tiếng Anh cho agent–agent, tiếng Việt cho agent–user — [Cần người dùng phê duyệt]).
5. Multi-turn resume cho pause/handoff phụ thuộc `--resume` với `-p` (A05).

## 15. Acceptance Criteria

- [ ] Zod schema cho 10 message type; test validate payload hợp lệ/không hợp lệ.
- [ ] Deliver message khi spawn: test prompt inject chứa đúng N=10 message gần nhất và ghi `truncated`.
- [ ] `TASK_HANDOFF` không đổi owner cho tới khi `ACK(accepted)` + scheduler reassign (test).
- [ ] `REQUEST_REVIEW` tạo task con REVIEW, task cha vào REVIEW; `REVIEW_RESULT(PASS)` → cha DONE chỉ khi orchestrator xác nhận evidence (test).
- [ ] Context budget: inject vượt 8 000 token → cắt đúng thứ tự ưu tiên (test với fixture).
- [ ] Snapshot tái lập: với `executionId` bất kỳ, API trả về đúng prompt đã inject (sha khớp).
- [ ] Meeting 3 role giả lập (mock adapter) → minutes có decisions[] và action items; dissent → approval request.
- [ ] Thread depth > 6 bị chặn và tạo cảnh báo.
