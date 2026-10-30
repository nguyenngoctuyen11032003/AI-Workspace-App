# SRS — Software Requirements Specification: AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0) · **Tác giả:** Agent 12 — BA · **Trạng thái:** Draft · **Ngày:** 2026-10-30
**Cấu trúc tham khảo:** IEEE 830 / ISO 29148 (rút gọn)

> Quy ước: [Owner] = yêu cầu gốc; [Giả định] = BA suy diễn; [Đề xuất] = cần Owner chốt. Chi tiết FR tại `FUNCTIONAL_REQUIREMENTS.md`; NFR tại `NON_FUNCTIONAL_REQUIREMENTS.md`.

---

## 1. Giới thiệu

### 1.1 Mục đích
SRS mô tả hệ thống AI WORKSPACE ở mức đặc tả: ngữ cảnh, giao diện bên ngoài, mô hình dữ liệu khái niệm, hành vi chính (state machine, event), ràng buộc và giả định. Đây là tài liệu đầu vào cho Agent kiến trúc, thiết kế UI/UX, QA và Security ở các phase sau.

### 1.2 Phạm vi
Hệ thống phần mềm chạy trên máy người dùng (Windows 11, single-user), điều phối nhiều AI agent thông qua provider CLI, thực thi công việc phần mềm thật trong sandbox, trực quan hóa bằng Virtual Office 2D và dashboard.

**Trong phạm vi:** 12 nhóm năng lực [Owner]. **Ngoài phạm vi:** dịch vụ cloud/SaaS đa người dùng (tương lai), marketplace, thanh toán.

### 1.3 Định nghĩa, thuật ngữ

| Thuật ngữ | Định nghĩa |
|---|---|
| Owner / Director | Người dùng duy nhất trong MVP; quyền cao nhất; nghiệm thu cuối. |
| Company / Workspace | Đơn vị tổ chức chứa phòng ban, agent, project, skill. |
| Department | Phòng ban; ánh xạ tới một phòng trong Virtual Office. |
| Agent | Thực thể AI có cấu hình (provider, model, prompt, skill...) thực thi task. |
| Provider | Nguồn mô hình AI truy cập qua adapter. MVP: Claude (OWR-2 Q-17). Codex hoãn khỏi MVP (D-005), OpenHands hoãn đến sau Phase 4 (D-006); generic CLI chỉ là mở rộng qua `AgentAdapter`. |
| Adapter | Lớp trừu tượng hóa provider với 13 operation chuẩn. |
| Session | Phiên làm việc với provider (có thể resume). |
| Project | Dự án phần mềm được tạo từ prompt; có workspace/repo riêng. |
| Workflow | Chuỗi node/giai đoạn được engine bền vững thực thi. |
| Stage | Giai đoạn trong template SDLC (8 giai đoạn). |
| Task | Đơn vị công việc gán cho agent; có 10 trạng thái. |
| Event | Bản ghi có cấu trúc về sự thay đổi trong hệ thống; nguồn duy nhất cho UI. |
| Skill | Gói kiến thức theo chuẩn Agent Skills (`SKILL.md` + references/examples/scripts). |
| Sandbox | Môi trường thực thi cô lập (Docker). |
| Approval gate | Node dừng chờ quyết định của con người. |
| Revision | Vòng sửa đổi sau reject; được đánh phiên bản. |
| Emergency stop | Dừng khẩn cấp toàn bộ hoạt động agent. |

### 1.4 Tài liệu tham chiếu
`PRODUCT_VISION.md`, `PRD.md`, `FUNCTIONAL_REQUIREMENTS.md`, `NON_FUNCTIONAL_REQUIREMENTS.md`, `USER_ROLES_AND_PERMISSIONS.md`, `USER_FLOWS.md`, `ACCEPTANCE_CRITERIA.md`; các tài liệu `docs/architecture/`, `docs/security/`, `docs/design/` (do agent khác soạn).

## 2. Mô tả tổng quan

### 2.1 Bối cảnh hệ thống (System Context)

```
+-------------------+        +---------------------------+        +----------------------+
|  Owner (human)    | <----> |       AI WORKSPACE        | <----> | Provider CLIs        |
|  Desktop UI       |        |  - Company/Agent/Skill    |        |  Claude Code, Codex, |
|  (22 screens,     |        |  - Workflow Engine        |        |  OpenHands, generic  |
|   Virtual Office) |        |  - Event Bus / Message Bus|        +----------------------+
+-------------------+        |  - Resource Manager       |
                             |  - Provider Hub (adapters)| <----> | Docker (sandbox)     |
                             |  - Security & Audit       |        | Git (repos/worktrees)|
                             |  - Persistence (local DB) |        | OS Credential Manager|
                             +---------------------------+        +----------------------+
```

### 2.2 Các thành phần chính (conceptual)

1. **Presentation**: 22 màn hình + Virtual Office 2D render (canvas). Chỉ tiêu thụ event và query read-model.
2. **Company & Agent Service**: CRUD company, department, agent; lifecycle agent.
3. **Skill Service**: Skill Library, Teach Agent pipeline, loader skill theo task.
4. **Provider Hub**: adapter registry; 13 operation chuẩn; streaming event; usage.
5. **Workflow Engine (durable)**: node executor, checkpoint store, retry, parallel/merge, approval wait.
6. **Orchestrator / Resource Manager**: concurrency slot, queue, PM scoring, handover.
7. **Message Bus & Event Store**: persist event/message; publish tới UI.
8. **Sandbox Runner**: tạo/giới hạn/hủy container; allow/deny; audit.
9. **Persistence**: local DB (loại do kiến trúc quyết định) + file system (skills, workspaces).

### 2.3 Lớp người dùng
Xem `USER_ROLES_AND_PERMISSIONS.md`. MVP: 1 Owner. Tương lai: Project Owner, Viewer [Giả định].

### 2.4 Môi trường vận hành
- Windows 11 (Owner); [Giả định] macOS/Linux ở phase sau.
- Yêu cầu: Claude Code CLI / Codex CLI đã login; Docker Desktop; Git.
- [Chưa chốt] Shell ứng dụng: Tauri / Electron / web local (OI-03).

### 2.5 Ràng buộc thiết kế
- C-01 Không API key bắt buộc; dùng CLI auth [Owner].
- C-02 Không lưu credential plaintext; không gửi credential tới frontend; không log credential [Owner].
- C-03 Event là nguồn duy nhất cho UI [Owner].
- C-04 Không auto-deploy production [Owner].
- C-05 Skill theo chuẩn thư mục Agent Skills [Owner].
- C-06 Concurrency mặc định 5, tối đa 18 [Owner].
- C-07 Dữ liệu dashboard phải thật [Owner].

### 2.6 Giả định và phụ thuộc
- A-01 CLI provider có thể được gọi theo chế độ non-interactive/headless và trả output có thể parse (JSON/stream). [Giả định — cần xác minh kỹ thuật]
- A-02 CLI provider hỗ trợ resume session theo id. [Giả định]
- A-03 Docker Desktop có thể được điều khiển từ ứng dụng (Docker Engine API/CLI). [Giả định]
- A-04 Mỗi project là một git repo; agent dùng worktree. [Giả định]

## 3. Giao diện bên ngoài

### 3.1 Giao diện người dùng
22 màn hình [Owner] (danh sách tại `PRD.md` §5.11). Yêu cầu chung: điều hướng nhất quán, trạng thái realtime, không có dữ liệu giả.

### 3.2 Giao diện phần mềm — Provider Adapter (contract khái niệm)

| Operation | Input (khái niệm) | Output | Ghi chú |
|---|---|---|---|
| Connect | providerId, config | ConnectionStatus | Phát hiện CLI, kiểm tra login |
| Disconnect | providerId | ok | Đóng session đang mở |
| Authenticate | providerId | AuthStatus | Chỉ hướng dẫn người dùng login qua CLI; không nhận mật khẩu |
| CheckConnection | providerId | ConnectionStatus | Dùng cho heartbeat |
| ListModels | providerId | Model[] | |
| CreateSession | agentId, workspace, options | sessionId | |
| ResumeSession | sessionId | ok / NOT_SUPPORTED | |
| SendMessage | sessionId, message, attachments | turnId | |
| StreamEvents | sessionId | stream<ProviderEvent> | Chuẩn hóa thành event hệ thống |
| CancelTask | sessionId / turnId | ok | Kill tiến trình nếu cần |
| GetUsage | sessionId / range | Usage (tokens, cost, isEstimated) | |
| GetError | sessionId | ErrorInfo | |
| GetSessionHistory | sessionId | Turn[] | |

Lỗi chuẩn [Đề xuất]: NOT_INSTALLED, NOT_AUTHENTICATED, SESSION_EXPIRED, RATE_LIMITED, NOT_SUPPORTED, TIMEOUT, PROCESS_CRASHED.

### 3.3 Giao diện phần mềm — Sandbox Runner
- Tạo container từ image chuẩn với mount workspace của task (read/write giới hạn), giới hạn CPU/RAM/disk/time, network policy.
- Thực thi lệnh với kiểm tra allow/deny; trả stdout/stderr/exit code; ghi audit.
- Hủy container khi task xong/emergency stop.

### 3.4 Giao diện phần mềm — Git
- Tạo branch/worktree per agent/task; commit theo task; mở PR/merge request nội bộ cho code review; merge sau khi review pass.

### 3.5 Giao diện phần mềm — OS Credential Manager
- Chỉ dùng khi cần lưu secret (API key fallback [chưa chốt]).

### 3.6 Giao diện truyền thông
- Nội bộ: event bus/message bus (in-process hoặc IPC); UI nhận event qua subscription (WebSocket/IPC tùy shell).

## 4. Mô hình dữ liệu khái niệm

```
Company 1---* Department 1---* Agent
Company 1---* Project 1---* Workflow 1---* Stage 1---* Task
Agent *---* Skill (AgentSkill: version, assignedAt)
Agent 1---* Session (provider)
Task 1---* Task (subtask) ; Task 1---* FileOwnership ; Task 1---* Artifact
Project 1---* Approval ; Approval *---1 Stage/Task
Project 1---* Bug ; Bug 1---1 Task (fix)
Project 1---* Revision (version n) 
* ---> Event (append-only) ; * ---> Message (persisted) ; * ---> AuditLog
Workflow 1---* Checkpoint
Provider 1---* Model ; Provider 1---* UsageRecord
OfficeLayout 1---* Room 1---* Desk ; Agent 1---1 Desk
```

Thực thể chính và trường tiêu biểu:

- **Agent**: id, name, avatar, jobTitle, departmentId, description, responsibilities, systemPrompt, providerId, model, skills[], tools[], memoryConfig, permissionLevel, workingDir, maxTokenBudget, maxExecTime, maxRetry, reportingManagerId, status(12), roleTemplate, createdAt.
- **Task**: id, projectId, workflowId, stageId, title, description, assigneeId, status(10), priority, dependencies[], inputs, outputs[], fileOwnership[], tokenBudget, timeBudget, retryCount, parentId, version, assignmentMode(AUTO/MANUAL), assignmentReason.
- **Event**: id, type, ts, actorType(agent/human/system), actorId, entityType, entityId, payload, correlationId, projectId.
- **Skill**: id, name, category, description, version, path(`skills/<name>/`), mandatoryRules[], allowedTools[], checklists[], tests[].
- **Approval**: id, projectId, stageId/taskId, type, status, requestedBy, decidedBy, decision, feedback, markups[], createdAt, decidedAt.
- **Checkpoint**: id, workflowId, nodeId, state(blob), ts.

## 5. Hành vi hệ thống

### 5.1 State machine — Agent (12 trạng thái) [Owner]; chuyển tiếp [Đề xuất]

| Từ | Tới | Trigger (event) |
|---|---|---|
| Created | Idle | AGENT_CREATED hoàn tất, chưa kết nối provider |
| Idle | Available | Provider Connected & agent Enabled |
| Available | Assigned | TASK_ASSIGNED |
| Assigned | Running | TASK_STARTED |
| Running | Waiting | chờ approval / input / agent khác |
| Running | Blocked | thiếu tài nguyên, lỗi phụ thuộc, pause-by-policy |
| Running | Reviewing | task sang REVIEWING (agent là reviewer hoặc chờ review) |
| Running | Completed | TASK_COMPLETED |
| Running | Failed | TASK_FAILED (hết retry) |
| Completed/Failed | Available | hệ thống giải phóng agent sau khi ghi nhận kết quả |
| bất kỳ (trừ Disabled) | Paused | Owner Pause |
| Paused | trạng thái trước | Owner Resume |
| bất kỳ | Disabled | Owner Disable |
| Disabled | Idle | Owner Enable |

> OI: Completed/Failed là trạng thái tức thời của agent gắn với task cuối; cần Owner xác nhận cách hiểu (OI-PRD-05/06).

### 5.2 State machine — Task (10 trạng thái) [Owner]; chuyển tiếp [Đề xuất]

BACKLOG → READY (dependencies thỏa) → ASSIGNED (AUTO/MANUAL) → RUNNING → {COMPLETED | FAILED | BLOCKED | WAITING_APPROVAL | REVIEWING}; REVIEWING → COMPLETED | RUNNING (sửa); WAITING_APPROVAL → RUNNING/COMPLETED (approve) | RUNNING với revision (reject); FAILED → ASSIGNED (handover/retry) | CANCELLED; bất kỳ → CANCELLED (Owner).

### 5.3 Workflow template SDLC 8 giai đoạn [Owner]

| # | Giai đoạn | Agent chính | Approval | Output |
|---|---|---|---|---|
| 1 | Requirement Intake | PM | — | Project, team, backlog sơ bộ |
| 2 | Business Analysis | BA | **Có** | Tài liệu yêu cầu |
| 3 | System Design | Architect | — ([Đề xuất] tùy policy) | Thiết kế hệ thống |
| 4 | UI/UX Design | UI/UX | **Có** | Thiết kế UI |
| 5 | Development (parallel FE/BE/DB) | FE, BE, DB | — | Code trên branch riêng, đã code review |
| 6 | QA & Security & Code Review & Product Evaluation | QA, Security, Reviewer, Evaluator | — | Bug records, báo cáo |
| 7 | Build & Deployment preview | DevOps/Release | — | Preview chạy được; không prod |
| 8 | Human Acceptance | Owner | **Có** | Approve / Reject + revision tasks |

Reject ở giai đoạn 8: tạo revision tasks targeted; tăng version; quay lại giai đoạn 5–7 chỉ với phạm vi liên quan; không chạy lại 1–4 [Owner].

### 5.4 Event-driven UI
Mọi thay đổi → Event Store (append-only) → publish → UI/Virtual Office. Virtual Office ánh xạ event → animation/di chuyển theo bảng cấu hình (FR-162). Không có timer giả lập hoạt động.

### 5.5 Teach Agent pipeline
Nhận chỉ dẫn → Parse thành rules → Gợi ý skill → (Owner review/sửa/xóa) → Lưu vào library → Gán agent → Verify đã load → Evaluation. Không bước nào tự lưu khi chưa review [Owner].

### 5.6 Emergency stop
Một hành động → EMERGENCY_STOP event → CancelTask mọi session → dừng/hủy container → task RUNNING → BLOCKED → agent → Paused → UI hiển thị trạng thái dừng khẩn cấp → Owner resume từng phần hoặc toàn bộ.

### 5.7 Recovery sau restart
Khởi động → đọc checkpoint/event store → dựng lại read-model → với workflow đang chạy: tiếp tục từ checkpoint (nếu provider session resume được) hoặc đặt WAITING cho Owner xác nhận [Đề xuất: mặc định chờ xác nhận để tránh chi phí không mong muốn].

## 6. Yêu cầu phi chức năng (tham chiếu)
Xem `NON_FUNCTIONAL_REQUIREMENTS.md` (NFR-001 …).

## 7. Yêu cầu khác
- Ghi log có cấu trúc, mask secret.
- Dữ liệu người dùng lưu local; không gửi telemetry nếu không bật [Đề xuất].

## 8. Dependencies
Claude Code CLI, Codex CLI, Docker Desktop, Git, OS Credential Manager; quyết định kiến trúc shell, DB, event bus; asset pixel-art.

## 9. Rủi ro
- CLI không headless/không parse được → adapter mong manh.
- Session resume không được hỗ trợ → recovery phải chạy lại turn.
- Docker trên Windows (WSL2) hiệu năng và quyền truy cập file.
- Khối lượng event lớn → UI chậm nếu không batch.

## 10. Vấn đề chưa giải quyết / mâu thuẫn

| ID | Vấn đề |
|---|---|
| OI-SRS-01 | **Host vs sandbox:** CLI provider chạy ở host (nơi có login) hay trong container? Nếu ở host, phần "thực thi code" của agent (chạy test, build) ở sandbox — nhưng chính CLI (Claude Code/Codex) tự chạy lệnh/sửa file; làm sao chặn CLI chạy lệnh ngoài sandbox? [Cần kiến trúc: hooks/permission mode của CLI, hoặc chạy CLI trong container với credential mount tạm thời — cả hai đều có đánh đổi bảo mật]. |
| OI-SRS-02 | **18 agent vs rate limit:** cần chính sách khi RATE_LIMITED. |
| OI-SRS-03 | **Tauri vs web:** quyết định ảnh hưởng IPC, credential manager, Docker control. |
| OI-SRS-04 | Idle vs Available; Completed/Failed cho agent. |
| OI-SRS-05 | ResumeSession có khả thi với từng CLI? |
| OI-SRS-06 | Preview ở stage 7 chạy ở đâu (container port mapping?). |
| OI-SRS-07 | Merge conflict giữa FE/BE/DB branches do ai giải quyết (Integration agent?). |
| OI-SRS-08 | Nguồn pricing để tính cost khi dùng CLI subscription. |

## 11. Cần người dùng phê duyệt

| # | Hạng mục |
|---|---|
| AP-SRS-01 | Thuật ngữ §1.3 |
| AP-SRS-02 | Ràng buộc C-01…C-07 và giả định A-01…A-04 |
| AP-SRS-03 | Contract adapter §3.2 và bộ lỗi chuẩn |
| AP-SRS-04 | State machine agent §5.1 và task §5.2 |
| AP-SRS-05 | Hành vi recovery mặc định (chờ xác nhận) §5.7 |
| AP-SRS-06 | Giải quyết OI-SRS-01…08 |
