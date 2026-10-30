# PRD — Product Requirements Document: AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0 — Documentation only)
**Tác giả:** Agent 12 — Business Analyst
**Trạng thái:** Draft — chờ Owner phê duyệt
**Ngày:** 2026-10-30

> Quy ước: **[Owner]** = yêu cầu gốc; **[Giả định]** = giả định BA; **[Đề xuất]** = đề xuất BA cần Owner quyết. Mã FR-xxx tham chiếu `FUNCTIONAL_REQUIREMENTS.md`; AC-xxx tham chiếu `ACCEPTANCE_CRITERIA.md`.

---

## 1. Mục đích

PRD mô tả **cái gì** sản phẩm phải làm được từ góc nhìn người dùng và nghiệp vụ, theo từng subsystem, kèm phạm vi MVP, luồng chính, màn hình, và các quyết định cần Owner chốt. PRD là cầu nối giữa `PRODUCT_VISION.md` (tại sao) và `SRS.md` / `FUNCTIONAL_REQUIREMENTS.md` (chi tiết kỹ thuật, có thể kiểm thử).

## 2. Phạm vi

- **Trong phạm vi:** 12 nhóm năng lực do Owner liệt kê; 22 màn hình; tiêu chí MVP A–G.
- **Ngoài phạm vi:** kiến trúc, công nghệ, thiết kế hình ảnh, kế hoạch triển khai, giá bán.

## 3. Bối cảnh & mục tiêu sản phẩm

**[Owner]** AI WORKSPACE là nền tảng quản lý, điều phối nhiều AI agent mô phỏng một công ty phần mềm thực. Người dùng là Owner/Product Owner/Director, nghiệm thu cuối. Agent làm việc thật. Không phải chatbot, không chỉ dashboard.

**Mục tiêu sản phẩm (Product Goals):**
- G1. Từ một yêu cầu tự nhiên tạo ra một dự án phần mềm được thiết kế, lập trình, kiểm thử, rà soát bảo mật và có bản preview để nghiệm thu.
- G2. Cho người dùng nhìn thấy và điều khiển toàn bộ "công ty" theo thời gian thực qua Virtual Office 2D và dashboard.
- G3. Tích lũy kỹ năng/kiến thức có kiểm soát qua Skill Library.
- G4. An toàn, bền vững, minh bạch chi phí.

## 4. Người dùng & vai trò (tóm tắt)

Xem `USER_ROLES_AND_PERMISSIONS.md`. MVP: single-user desktop, một người đóng vai Owner/Director. **[Giả định]** Tương lai: Project Owner, Viewer.

## 5. Yêu cầu theo subsystem

### 5.1 AI Company Management (FR-001…FR-020)

**[Owner]**
- Multi-company / multi-workspace.
- Phòng ban mặc định (12): Management, Business Analysis, System Architecture, UI/UX Design, Frontend, Backend, Database, QA, Cybersecurity, DevOps, Code Review, Product Evaluation.
- Người dùng thêm phòng ban tùy ý, ví dụ: Mobile, Data Engineering, AI Research, SEO, Marketing, Technical Writing.
- Org chart kéo-thả (drag-drop), trưởng phòng (department head), cấp quản lý (management levels), ngân sách phòng ban, hiệu suất phòng ban.

**Câu chuyện người dùng (User Stories):**
- US-1.1 Là Owner, tôi muốn tạo công ty với 12 phòng ban mặc định để bắt đầu nhanh.
- US-1.2 Là Owner, tôi muốn thêm/sửa/xóa phòng ban và chỉ định trưởng phòng.
- US-1.3 Là Owner, tôi muốn kéo-thả agent giữa các phòng ban trên org chart và thấy quan hệ báo cáo thay đổi.
- US-1.4 Là Owner, tôi muốn đặt ngân sách (token/cost) cho phòng ban và thấy mức sử dụng so với ngân sách.

**[Đề xuất]** MVP chỉ cần 1 company/workspace; multi-company ở Phase 2 (cần Owner chốt — xem mục 11).

### 5.2 Dynamic Agent Management (FR-021…FR-050)

**[Owner] Agent Builder** — form với các trường: name, avatar, job title, department, description, responsibilities, system prompt, provider, model, skills, tools, memory config, permission level, working directory, max token budget, max execution time, max retry, reporting manager, status.

**[Owner] Lifecycle states (12):** Created, Idle, Available, Assigned, Running, Waiting, Blocked, Reviewing, Completed, Failed, Paused, Disabled — đồng bộ qua event.

**[Owner] Agent Profile:** thông tin, tasks, activity, lịch sử giao tiếp, token usage, cost, performance, errors, permissions, memory; nút Chat / Assign / Pause / Resume / Disable.

**[Owner] Cộng tác (collaboration):** nhắn tin cho agent khác, bàn giao tài liệu (handoff), yêu cầu trợ giúp, tạo subtask, báo cáo cho manager, đề xuất thay đổi, họp (meetings). Không sửa code của agent khác nếu không được phép. Giao tiếp quan trọng được lưu bền (persisted).

**User Stories:**
- US-2.1 Là Owner, tôi tạo agent mới bằng form, chọn provider/model, gán skill, đặt giới hạn token/thời gian/retry.
- US-2.2 Là Owner, tôi xem hồ sơ agent với tab Overview/Profile/Skills/Knowledge/Provider/Tasks/Conversations/Activity Logs/Performance/Permissions.
- US-2.3 Là Owner, tôi Pause/Resume/Disable agent và thấy trạng thái thay đổi ngay trong Virtual Office.
- US-2.4 Là Owner, tôi chat trực tiếp với một agent từ profile.
- US-2.5 Là agent, tôi gửi handoff doc cho agent khác và tạo subtask khi cần.

### 5.3 Skill Management (FR-051…FR-075)

**[Owner]**
- Skill Library với danh mục: Development, Architecture, Database, Testing, Security, DevOps (kèm ví dụ).
- Custom skill: name, description, instructions, upload MD/PDF/references, code examples, checklists, mandatory rules, allowed tools, test, versioning.
- Chuẩn Agent Skills: `skills/<name>/SKILL.md`, `references/`, `examples/`, `scripts/`.
- Teach Agent flow: nhận chỉ dẫn → parse thành rules → gợi ý skill → lưu vào library → gán → verify đã load → chạy evaluation. Kiến thức phải review/sửa/xóa được, không tự động persist.
- Gán skill many-to-many; chỉ load skill liên quan cho mỗi task.

**User Stories:**
- US-3.1 Là Owner, tôi duyệt Skill Library theo category và xem ví dụ.
- US-3.2 Là Owner, tôi tạo custom skill, upload tài liệu tham khảo, viết checklist và mandatory rules, đặt phiên bản.
- US-3.3 Là Owner, tôi "dạy" agent bằng một đoạn chỉ dẫn; hệ thống đề xuất skill; tôi review/sửa rồi mới lưu.
- US-3.4 Là Owner, tôi gán một skill cho nhiều agent và kiểm tra agent đã load skill.

### 5.4 AI Provider Hub (FR-076…FR-100)

**[Owner]**
- Adapter: Claude Code, Codex (CLI-based, dùng lại login CLI hiện có; không cần API key nếu CLI auth hoạt động), OpenHands, generic CLI.
- Standard operations: Connect, Disconnect, Authenticate, CheckConnection, ListModels, CreateSession, ResumeSession, SendMessage, StreamEvents, CancelTask, GetUsage, GetError, GetSessionHistory.
- Luồng kết nối với hướng dẫn cài đặt/login.
- Luật bảo mật xác thực: không hỏi mật khẩu; không trích xuất token; không lưu OAuth plaintext trong DB; không gửi credential lên frontend; dùng OS credential manager nếu cần; logout/reconnect/expiry; không credential trong log; không chia sẻ credential giữa agent.
- Provider dashboard.

**User Stories:**
- US-4.1 Là Owner, tôi kết nối Claude Code; nếu chưa cài/login, hệ thống hướng dẫn từng bước.
- US-4.2 Là Owner, tôi thấy trạng thái kết nối, model khả dụng, usage của từng provider.
- US-4.3 Là Owner, tôi ngắt kết nối/kết nối lại khi phiên hết hạn.

### 5.5 Orchestration & Workflow (FR-101…FR-150)

**[Owner]**
- Durable workflow engine: checkpoint, retry, branching, parallel, human approval.
- Task đầy đủ trường; AUTO assignment (PM scoring: role, skills, tools, workload, provider availability, permissions) và MANUAL.
- Task handover khi thất bại.
- Workflow Builder node-based: Agent, Approval, Condition, Parallel, Merge, Tool, Test, Deployment nodes; lưu template; dry-run.
- Event system có cấu trúc (AGENT_CREATED … HUMAN_REJECTED) là nguồn duy nhất cho UI/office.
- Template mặc định 8 giai đoạn "FULL SOFTWARE DEVELOPMENT LIFECYCLE":
  1. Requirement Intake
  2. Business Analysis (có approval)
  3. System Design
  4. UI/UX Design (có approval)
  5. Development song song FE/BE/DB với git branch và code review
  6. QA & Security & Code Review & Product Evaluation (tạo bug record)
  7. Build & Deployment preview (không auto deploy production)
  8. Human Acceptance: approve/reject/feedback/screenshot markup; reject tạo revision task có mục tiêu, không chạy lại toàn bộ; revision được đánh phiên bản.

**User Stories:**
- US-5.1 Là Owner, tôi nhập prompt; hệ thống tạo project và khởi chạy template SDLC.
- US-5.2 Là Owner, tôi phê duyệt tài liệu BA và thiết kế UI/UX tại Approval Center trước khi dev bắt đầu.
- US-5.3 Là Owner, tôi nghiệm thu bản preview, đánh dấu lỗi trên screenshot và reject; chỉ các task sửa lỗi liên quan được tạo.
- US-5.4 Là Owner, tôi dựng workflow riêng bằng Workflow Builder, lưu template và dry-run.

### 5.6 Virtual Office 2D (FR-151…FR-180)

**[Owner]**
- Phòng: Reception, Management, BA, Architecture, Design Studio, FE, BE, DB, QA, Security, DevOps, Meeting, Relax, Coffee (14).
- Zoom/pan, floors, edit layout, furniture.
- Nhân vật: sprite, outfit, desk, position, animation (Idle, Walk, Run, Sit, Work, Typing, Reading, Thinking, Meeting, Talking, Celebrating, Error, Waiting — 13), speech bubble.
- Di chuyển chỉ do event thật; click-to-interact.

**User Stories:**
- US-6.1 Là Owner, tôi thấy agent đi từ bàn sang phòng Meeting khi event MEETING_STARTED phát sinh.
- US-6.2 Là Owner, tôi click vào nhân vật để mở nhanh profile/chat/assign.
- US-6.3 Là Owner, tôi zoom/pan và (Phase 2) chỉnh layout, đặt furniture.

### 5.7 Dashboards (FR-181…FR-195)

**[Owner]** Overview metrics; dashboard project/agent/workflow/cost/quality/system; chỉ dữ liệu thật.

### 5.8 Approval Center (FR-196…FR-205)

**[Owner]** Trung tâm phê duyệt với approval policies cấu hình được.

### 5.9 Security & Sandbox (FR-206…FR-230)

**[Owner]** Workspace per project; Docker sandbox; limits; allow/deny lists; hạn chế FS/network; credential isolation; audit logs; emergency stop.

### 5.10 Multi-agent Orchestration (FR-231…FR-260)

**[Owner]**
- Concurrency toàn cục mặc định 5, cấu hình tối đa 18 agent đồng thời.
- 18 vai trò mặc định: PM, BA, Architect, UI/UX, FE, BE, DB&Infra, QA, Security, Code Reviewer/Product Evaluator, R&D, Integration, Performance, DevSecOps, Documentation, UX Research, Test Automation, Release Manager.
- Workspace/git worktree per agent; file ownership per task.
- Task status: BACKLOG, READY, ASSIGNED, RUNNING, BLOCKED, WAITING_APPROVAL, REVIEWING, FAILED, COMPLETED, CANCELLED.
- Message bus; resource manager; monitoring dashboard với pause/resume/stop/retry/reassign/logs/changes/open workspace/request review.

### 5.11 Premium UI — 22 màn hình (FR-261…FR-285)

**[Owner]** Welcome/Onboarding, Main Dashboard, Virtual Office, Org Chart, Department Management, Agent Management, Agent Detail (10 tab), Agent Builder, Skill Library, Skill Editor, Provider Hub, Project Management, Project Detail, Workflow Builder, Task Board, Activity Center, Approval Center, Code Workspace, Product Preview, QA & Security Report, Cost Management, Settings.

### 5.12 Reliability (FR-286…FR-300)

**[Owner]** Không mất state khi restart; retry; checkpoint; recovery.

## 6. Luồng chính (tóm tắt)

Chi tiết tại `USER_FLOWS.md`. Các luồng cốt lõi: Onboarding & kết nối provider; Tạo agent; Dạy agent; Tạo project từ prompt → nghiệm thu; Reject → revision; Emergency stop; Pause/Resume agent; Tạo custom skill; Workflow Builder; Theo dõi chi phí.

## 7. Màn hình & điều hướng (tổng quan)

**[Đề xuất]** Điều hướng chính (sidebar): Dashboard → Virtual Office → Projects → Agents → Skills → Workflows → Approvals → Activity → Providers → Cost → Settings. Org Chart & Departments là tab trong "Company". Code Workspace, Product Preview, QA & Security Report là tab trong Project Detail. Owner cần chốt (AP-PRD-06).

## 8. Phạm vi MVP (đối chiếu tiêu chí A–G)

| Nhóm | Tiêu chí MVP [Owner] | Subsystem liên quan |
|---|---|---|
| A — Agent | Tạo/sửa/xóa agent; lifecycle sync; profile; pause/resume/disable | 5.2 |
| B — Provider | Kết nối Claude (Claude Agent SDK, OWR-2 Q-17) qua adapter; standard ops hoạt động; luật bảo mật. **Codex hoãn khỏi MVP (OWR-2 D-005), OpenHands hoãn đến sau Phase 4 (D-006)**; giữ abstraction `AgentAdapter`. [Đã sửa: trước đây ghi "Claude Code & Codex"; FR-xxx liên quan do agent khác đồng bộ] | 5.4 |
| C — Workflow | Template SDLC 8 giai đoạn chạy end-to-end với approval gate; reject → revision | 5.5, 5.8 |
| D — Skill | Library, custom skill, Teach Agent, gán & verify | 5.3 |
| E — Virtual Office | 14 phòng, nhân vật, animation theo event thật, click-to-interact | 5.6 |
| F — Security | Sandbox, allow/deny, credential isolation, audit, emergency stop | 5.9 |
| G — Reliability | Restart không mất state; retry; checkpoint; recovery | 5.12 |

**[Đề xuất]** Ngoài MVP: OpenHands/generic CLI adapter, multi-company, Workflow Builder dry-run đầy đủ, floors/edit layout/furniture, 22 màn hình "premium polish" (MVP có thể là bản functional của ~15 màn hình cốt lõi).

## 9. Dependencies

- Claude Code CLI, Codex CLI (đã login) — bên ngoài.
- Docker Desktop, Git — bên ngoài.
- Quyết định kiến trúc shell (Tauri/web) từ `docs/architecture/`.
- Bộ sprite/pixel-art asset từ `docs/design/`.
- Mô hình sandbox/credential từ `docs/security/`.

## 10. Rủi ro

| ID | Rủi ro | Ảnh hưởng |
|---|---|---|
| R-PRD-01 | Phạm vi MVP không được chốt → ước lượng sai | Lịch trình |
| R-PRD-02 | Provider CLI không cung cấp đủ standard ops (vd. GetUsage, ResumeSession) qua CLI | Adapter phải "giả lập" hoặc giảm tính năng |
| R-PRD-03 | PM scoring tự động phân công sai → chất lượng thấp | Cần MANUAL override và log lý do |
| R-PRD-04 | Rate limit với 18 agent | Hiệu năng thực tế thấp hơn kỳ vọng |
| R-PRD-05 | Pixel-art asset chưa có | Virtual Office trễ |
| R-PRD-06 | Event schema thay đổi giữa chừng | UI/office vỡ |

## 11. Vấn đề chưa giải quyết / mâu thuẫn

| ID | Vấn đề |
|---|---|
| OI-PRD-01 | Host execution vs Docker sandbox: CLI login nằm ở host, sandbox cô lập FS/network — adapter chạy ở đâu? (trùng OI-01 Vision) |
| OI-PRD-02 | 18 agent song song vs rate limit 1 tài khoản CLI. |
| OI-PRD-03 | Tauri desktop vs web app — chưa chốt. |
| OI-PRD-04 | "Max concurrency 18" và "18 vai trò mặc định" có phải cùng một con số cố ý? Nếu người dùng thêm vai trò thứ 19, giới hạn có tăng? |
| OI-PRD-05 | Trạng thái agent "Completed"/"Failed" là trạng thái của agent hay của task? Agent sau khi task xong nên về Idle/Available; cần định nghĩa chuyển tiếp. |
| OI-PRD-06 | Lifecycle có cả "Idle" và "Available" — khác nhau thế nào? [Giả định] Idle = không có task, chưa sẵn sàng (vd. provider chưa kết nối); Available = sẵn sàng nhận task. |
| OI-PRD-07 | "Meetings" giữa agent: là cơ chế thật (multi-agent conversation) hay chỉ animation? |
| OI-PRD-08 | "Memory config" của agent: phạm vi (per project? global?), cách lưu, cách xóa — chưa định nghĩa. |
| OI-PRD-09 | "Budgets" phòng ban tính theo token hay tiền (USD)? Tỷ giá/pricing lấy từ đâu khi dùng CLI subscription (không có giá per token)? |
| OI-PRD-10 | Stage 5 "Development parallel FE/BE/DB với git branches": merge strategy khi conflict ai xử lý? |
| OI-PRD-11 | Stage 7 "Build & Deployment preview": preview chạy ở đâu (local container? port?) |
| OI-PRD-12 | Code Reviewer/Product Evaluator — 1 hay 2 vai trò. |

## 12. Cần người dùng phê duyệt

| # | Hạng mục |
|---|---|
| AP-PRD-01 | Danh sách subsystem và mapping FR range (mục 5) |
| AP-PRD-02 | Phạm vi MVP (mục 8) và các hạng mục dời Phase 2 |
| AP-PRD-03 | Giải đáp OI-PRD-01 … OI-PRD-12 |
| AP-PRD-04 | Định nghĩa Idle vs Available (OI-PRD-06) |
| AP-PRD-05 | Đơn vị ngân sách (token / USD) |
| AP-PRD-06 | Cấu trúc điều hướng 22 màn hình (mục 7) |
| AP-PRD-07 | Ngôn ngữ UI |

---
*Tài liệu liên quan:* `PRODUCT_VISION.md`, `SRS.md`, `FUNCTIONAL_REQUIREMENTS.md`, `NON_FUNCTIONAL_REQUIREMENTS.md`, `USER_ROLES_AND_PERMISSIONS.md`, `USER_FLOWS.md`, `ACCEPTANCE_CRITERIA.md`.
