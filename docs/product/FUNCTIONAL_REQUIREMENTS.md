# FUNCTIONAL REQUIREMENTS — AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0) · **Tác giả:** Agent 12 — BA · **Trạng thái:** Draft · **Ngày:** 2026-10-30

> Quy ước:
> - **Priority (MoSCoW):** M = Must, S = Should, C = Could, W = Won't (lần này).
> - **Phase:** MVP (Phase 1), P2 (Phase 2), P3 (tương lai).
> - **Nguồn:** [Owner] = yêu cầu gốc; [Giả định] = BA suy diễn; [Đề xuất] = BA đề xuất, cần Owner chốt.
> - Dải mã FR theo subsystem: 001–020 Company · 021–050 Agent · 051–075 Skill · 076–100 Provider · 101–150 Workflow · 151–180 Virtual Office · 181–195 Dashboard · 196–205 Approval · 206–230 Security · 231–260 Multi-agent · 261–285 UI · 286–300 Reliability. Các số trống để dành cho bổ sung sau.

## 1. Mục đích

Liệt kê đầy đủ yêu cầu chức năng (FR) có mã định danh, ưu tiên, phase, nguồn và mô tả có thể kiểm thử. Mỗi FR là đơn vị truy vết đến `ACCEPTANCE_CRITERIA.md` (AC-xxx) và đến thiết kế/kiến trúc ở các phase sau.

## 2. Phạm vi

Toàn bộ 12 nhóm năng lực do Owner liệt kê. Không bao gồm NFR (xem `NON_FUNCTIONAL_REQUIREMENTS.md`).

## 3. Danh sách yêu cầu chức năng

### 3.1 AI Company Management (FR-001 … FR-020)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-001 | Hệ thống cho phép tạo một company/workspace với tên, mô tả, logo (tùy chọn). | M | MVP | [Owner] |
| FR-002 | Hệ thống hỗ trợ nhiều company/workspace và chuyển đổi giữa chúng; dữ liệu (agent, project, skill) cô lập theo workspace. | C | P2 | [Owner] đa workspace; [Đề xuất] phase P2 |
| FR-003 | Khi tạo company, hệ thống tự tạo 12 phòng ban mặc định: Management, Business Analysis, System Architecture, UI/UX Design, Frontend, Backend, Database, QA, Cybersecurity, DevOps, Code Review, Product Evaluation. | M | MVP | [Owner] |
| FR-004 | Người dùng tạo/sửa/xóa phòng ban (tên, mô tả, màu, icon). Ví dụ phòng thêm: Mobile, Data Engineering, AI Research, SEO, Marketing, Technical Writing. Không xóa được phòng còn agent nếu chưa chuyển agent đi. | M | MVP | [Owner]; ràng buộc xóa [Đề xuất] |
| FR-005 | Người dùng chỉ định một agent làm trưởng phòng (department head) cho mỗi phòng ban. | S | MVP | [Owner] |
| FR-006 | Hệ thống hỗ trợ cấp quản lý (management levels) — ví dụ Director → Department Head → Lead → Member — dùng cho reporting chain. | S | P2 | [Owner]; số cấp [Giả định] |
| FR-007 | Màn hình Org Chart hiển thị cây tổ chức: company → phòng ban → trưởng phòng → agent. | S | MVP | [Owner] |
| FR-008 | Org Chart hỗ trợ kéo-thả agent giữa phòng ban/manager; thao tác tạo event và cập nhật reporting manager. | S | P2 | [Owner] |
| FR-009 | Người dùng đặt ngân sách cho phòng ban (đơn vị token hoặc tiền — chưa chốt) và hệ thống cảnh báo khi đạt ngưỡng (ví dụ 80%, 100%). | S | P2 | [Owner]; ngưỡng [Đề xuất] |
| FR-010 | Hệ thống hiển thị hiệu suất phòng ban: số task hoàn thành/thất bại, thời gian trung bình, chi phí, tỷ lệ reject. | S | P2 | [Owner]; chỉ số cụ thể [Đề xuất] |
| FR-011 | Mỗi phòng ban ánh xạ tới một phòng trong Virtual Office (xem FR-163). | M | MVP | [Giả định] |

### 3.2 Dynamic Agent Management (FR-021 … FR-050)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-021 | Agent Builder là form với đầy đủ trường: name, avatar, job title, department, description, responsibilities, system prompt, provider, model, skills, tools, memory config, permission level, working directory, max token budget, max execution time, max retry, reporting manager, status. | M | MVP | [Owner] |
| FR-022 | Hệ thống validate form: name bắt buộc và duy nhất trong company; provider phải ở trạng thái Connected; model thuộc ListModels của provider; giới hạn số là số dương. | M | MVP | [Đề xuất] |
| FR-023 | Người dùng chọn avatar/sprite từ bộ có sẵn (pixel-art) để hiển thị trong Virtual Office. | S | MVP | [Owner] |
| FR-024 | Agent có đúng 12 trạng thái lifecycle: Created, Idle, Available, Assigned, Running, Waiting, Blocked, Reviewing, Completed, Failed, Paused, Disabled. | M | MVP | [Owner] |
| FR-025 | Mọi chuyển trạng thái agent chỉ xảy ra thông qua event (ví dụ AGENT_STARTED → Running); UI không tự đổi trạng thái. | M | MVP | [Owner] |
| FR-026 | Agent Detail có 10 tab: Overview, Profile, Skills, Knowledge, Provider, Tasks, Conversations, Activity Logs, Performance, Permissions. | M | MVP | [Owner] |
| FR-027 | Người dùng chat trực tiếp với agent từ profile; cuộc hội thoại được lưu trong tab Conversations. | M | MVP | [Owner] |
| FR-028 | Người dùng gán task cho agent từ profile (nút Assign) — chọn task READY hoặc tạo task mới. | M | MVP | [Owner] |
| FR-029 | Nút Pause tạm dừng agent (hủy/giữ phiên provider đang chạy, task → BLOCKED hoặc giữ RUNNING tùy chính sách); Resume tiếp tục từ checkpoint. | M | MVP | [Owner]; hành vi task [Đề xuất] |
| FR-030 | Nút Disable đưa agent về Disabled: không nhận task mới, task đang chạy được handover (FR-111). | M | MVP | [Owner] |
| FR-031 | Xóa agent: chỉ khi không có task RUNNING; lịch sử/giao tiếp được giữ lại (soft delete). | M | MVP | [Đề xuất] |
| FR-032 | Agent gửi message cho agent khác qua message bus; message được lưu. | M | MVP | [Owner] |
| FR-033 | Agent bàn giao tài liệu (handoff document) cho agent khác; tài liệu gắn với task. | S | MVP | [Owner] |
| FR-034 | Agent yêu cầu trợ giúp (request help) từ manager hoặc agent khác; tạo event AGENT_HELP_REQUESTED. | S | MVP | [Owner] |
| FR-035 | Agent tạo subtask thuộc task hiện tại; subtask hiển thị trên Task Board. | M | MVP | [Owner] |
| FR-036 | Agent báo cáo tiến độ/kết quả cho reporting manager (report to manager). | S | MVP | [Owner] |
| FR-037 | Agent đề xuất thay đổi (propose change) với phạm vi ngoài task của mình; đề xuất phải được manager hoặc Owner phê duyệt. | C | P2 | [Owner] |
| FR-038 | Agent tổ chức/tham gia meeting (nhiều agent trao đổi theo chủ đề); meeting có biên bản. | C | P2 | [Owner]; cơ chế [Chưa rõ, OI-PRD-07] |
| FR-039 | Agent không được sửa file thuộc ownership của task khác nếu không có permission; vi phạm bị chặn và ghi audit. | M | MVP | [Owner] |
| FR-040 | Giao tiếp quan trọng (handoff, report, propose, help, approval-related) được persist và xem lại được trong tab Conversations. | M | MVP | [Owner] |
| FR-041 | Hệ thống ghi nhận token usage, cost, số lần lỗi, thời gian thực thi cho từng agent và hiển thị trong tab Performance. | M | MVP | [Owner] |
| FR-042 | Memory config cho phép bật/tắt bộ nhớ, đặt phạm vi (per project / global) và cho phép xem/xóa nội dung memory. | S | MVP | [Owner]; phạm vi [Giả định] |
| FR-043 | Hệ thống cung cấp 18 template vai trò mặc định (xem FR-233) để tạo nhanh agent. | S | MVP | [Owner] |
| FR-044 | Trạng thái agent hiển thị đồng nhất trên Agent Management, Agent Detail, Task Board, Virtual Office. | M | MVP | [Owner] |

### 3.3 Skill Management (FR-051 … FR-075)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-051 | Skill Library có danh mục: Development, Architecture, Database, Testing, Security, DevOps; mỗi danh mục có skill ví dụ kèm sẵn. | M | MVP | [Owner] |
| FR-052 | Người dùng duyệt, tìm kiếm, lọc skill theo category, tag, agent đang dùng. | M | MVP | [Owner] |
| FR-053 | Custom skill gồm: name, description, instructions, references (upload MD/PDF/link), code examples, checklists, mandatory rules, allowed tools, test, version. | M | MVP | [Owner] |
| FR-054 | Upload file MD/PDF làm references; nội dung được lưu vào `references/` của skill. | S | MVP | [Owner] |
| FR-055 | Skill được lưu theo chuẩn Agent Skills: `skills/<name>/SKILL.md`, `references/`, `examples/`, `scripts/`. | M | MVP | [Owner] |
| FR-056 | Skill có phiên bản (semantic version hoặc số tăng); agent được gán phiên bản cụ thể hoặc "latest". | S | MVP | [Owner] |
| FR-057 | Người dùng chạy "test skill": gán skill cho agent thử nghiệm với prompt mẫu và xem kết quả. | S | P2 | [Owner] |
| FR-058 | Teach Agent bước 1: nhận chỉ dẫn (text hoặc file) từ người dùng. | M | MVP | [Owner] |
| FR-059 | Teach Agent bước 2: hệ thống parse chỉ dẫn thành các rule có cấu trúc. | M | MVP | [Owner] |
| FR-060 | Teach Agent bước 3: hệ thống gợi ý tạo skill mới hoặc cập nhật skill hiện có. | M | MVP | [Owner] |
| FR-061 | Người dùng review/sửa/xóa các rule trước khi lưu; không rule nào được lưu tự động. | M | MVP | [Owner] |
| FR-062 | Gán skill ↔ agent là quan hệ many-to-many. | M | MVP | [Owner] |
| FR-063 | Teach Agent bước 6: hệ thống verify agent đã load skill (ví dụ agent xác nhận lại tóm tắt rule). | M | MVP | [Owner] |
| FR-064 | Teach Agent bước 7: chạy evaluation với bộ test case của skill và báo pass/fail. | S | MVP | [Owner] |
| FR-065 | Khi thực thi task, hệ thống chỉ nạp skill liên quan (theo category/tag/role/task type) thay vì toàn bộ library. | M | MVP | [Owner]; tiêu chí chọn [Đề xuất] |
| FR-066 | Mandatory rules trong skill được chèn vào prompt với mức ưu tiên cao và được đánh dấu bắt buộc. | S | MVP | [Owner] |
| FR-067 | Allowed tools trong skill giới hạn công cụ agent được dùng khi áp dụng skill đó. | S | MVP | [Owner] |
| FR-068 | Tab Knowledge của agent hiển thị skill và rule đã học, cho phép xóa/gỡ. | M | MVP | [Owner] |

### 3.4 AI Provider Hub (FR-076 … FR-100)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-076 | Adapter Claude Code (CLI-based). | M | MVP | [Owner] |
| FR-077 | Adapter Codex (CLI-based). | M | MVP | [Owner] |
| FR-078 | Adapter OpenHands. | C | P2 | [Owner]; cơ chế chưa rõ |
| FR-079 | Adapter generic CLI (cấu hình lệnh, tham số, parser output). | S | P2 | [Owner] |
| FR-080 | Adapter CLI dùng lại phiên đăng nhập CLI hiện có của người dùng; không bắt buộc API key nếu CLI auth hoạt động. | M | MVP | [Owner] |
| FR-081 | Mọi adapter triển khai 13 operation chuẩn: Connect, Disconnect, Authenticate, CheckConnection, ListModels, CreateSession, ResumeSession, SendMessage, StreamEvents, CancelTask, GetUsage, GetError, GetSessionHistory. Operation không hỗ trợ phải trả về lỗi NOT_SUPPORTED rõ ràng. | M | MVP | [Owner]; NOT_SUPPORTED [Đề xuất] |
| FR-082 | Luồng kết nối: phát hiện CLI → kiểm tra login → nếu thiếu, hiển thị hướng dẫn cài đặt/login từng bước → kiểm tra lại. | M | MVP | [Owner] |
| FR-083 | Hệ thống phát hiện CLI chưa cài, sai phiên bản, hoặc chưa login và hiển thị trạng thái tương ứng. | M | MVP | [Owner] |
| FR-084 | Hệ thống không bao giờ yêu cầu người dùng nhập mật khẩu tài khoản provider. | M | MVP | [Owner] |
| FR-085 | Hệ thống không trích xuất token từ file/cấu hình của CLI. | M | MVP | [Owner] |
| FR-086 | Không lưu OAuth token/API key dạng plaintext trong DB. | M | MVP | [Owner] |
| FR-087 | Không gửi bất kỳ credential nào tới frontend. | M | MVP | [Owner] |
| FR-088 | Nếu phải lưu credential (vd. API key fallback), dùng OS credential manager (Windows Credential Manager). | S | MVP | [Owner] |
| FR-089 | Hỗ trợ logout, reconnect và xử lý hết hạn phiên (expiry) với thông báo cho người dùng. | M | MVP | [Owner] |
| FR-090 | Không ghi credential vào log; log được mask tự động. | M | MVP | [Owner] |
| FR-091 | Không chia sẻ credential giữa các agent; mỗi agent chỉ gọi provider thông qua adapter. | M | MVP | [Owner] |
| FR-092 | Provider dashboard: trạng thái kết nối, model, usage, lỗi gần nhất, số session đang mở. | M | MVP | [Owner] |
| FR-093 | ListModels trả về danh sách model; Agent Builder chỉ cho chọn model trong danh sách này. | M | MVP | [Owner] |
| FR-094 | StreamEvents chuyển output của provider thành event chuẩn để UI/office tiêu thụ. | M | MVP | [Owner] |
| FR-095 | CancelTask hủy phiên provider đang chạy trong thời gian giới hạn. | M | MVP | [Owner] |
| FR-096 | GetUsage thu thập token/cost; nếu CLI không cung cấp, hệ thống ước lượng và đánh dấu "ước lượng". | S | MVP | [Owner]; ước lượng [Đề xuất] |
| FR-097 | ResumeSession tiếp tục phiên sau pause/restart nếu provider hỗ trợ. | S | MVP | [Owner] |
| FR-098 | API key fallback khi CLI auth không khả dụng. | C | — | [Chưa chốt — OI-06] |

### 3.5 Orchestration & Workflow (FR-101 … FR-150)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-101 | Workflow engine bền vững (durable): trạng thái workflow được lưu bền sau mỗi bước. | M | MVP | [Owner] |
| FR-102 | Checkpoint sau mỗi node/step; có thể tiếp tục từ checkpoint gần nhất. | M | MVP | [Owner] |
| FR-103 | Retry node thất bại theo chính sách (max retry, backoff). | M | MVP | [Owner] |
| FR-104 | Branching: node Condition rẽ nhánh theo kết quả. | M | MVP | [Owner] |
| FR-105 | Parallel: nhiều nhánh chạy song song và Merge khi tất cả xong. | M | MVP | [Owner] |
| FR-106 | Human approval node: workflow dừng chờ Owner approve/reject; không timeout mặc định. | M | MVP | [Owner]; timeout [Đề xuất] |
| FR-107 | Task có đầy đủ trường: id, title, description, project, workflow/stage, assignee, status, priority, dependencies, inputs, outputs/artifacts, file ownership, token budget, time budget, retry count, parent task, version, created/updated. | M | MVP | [Owner] "full fields"; danh sách [Đề xuất] |
| FR-108 | Task có 10 trạng thái: BACKLOG, READY, ASSIGNED, RUNNING, BLOCKED, WAITING_APPROVAL, REVIEWING, FAILED, COMPLETED, CANCELLED. | M | MVP | [Owner] |
| FR-109 | AUTO assignment: PM agent chấm điểm ứng viên theo role, skills, tools, workload, provider availability, permissions; chọn điểm cao nhất; ghi lại lý do. | M | MVP | [Owner]; ghi lý do [Đề xuất] |
| FR-110 | MANUAL assignment: Owner chọn agent cho task; ghi đè AUTO. | M | MVP | [Owner] |
| FR-111 | Task handover khi thất bại: sau khi hết retry, task được chuyển cho agent khác đủ điều kiện kèm context/log. | M | MVP | [Owner] |
| FR-112 | Workflow Builder dạng node-based canvas (kéo-thả node, nối cạnh). | S | P2 | [Owner] |
| FR-113 | Hỗ trợ 8 loại node: Agent, Approval, Condition, Parallel, Merge, Tool, Test, Deployment. | S | P2 | [Owner] |
| FR-114 | Lưu workflow thành template để tái sử dụng. | S | MVP | [Owner] |
| FR-115 | Dry-run workflow: mô phỏng đường đi mà không gọi provider. | C | P2 | [Owner] |
| FR-116 | Event system có cấu trúc: mỗi event có id, type, timestamp, actor, entity, payload, correlation id. | M | MVP | [Owner]; schema [Đề xuất] |
| FR-117 | Danh mục event tối thiểu: AGENT_CREATED, AGENT_UPDATED, AGENT_STATE_CHANGED, TASK_CREATED, TASK_ASSIGNED, TASK_STARTED, TASK_PROGRESS, TASK_BLOCKED, TASK_COMPLETED, TASK_FAILED, TASK_HANDOVER, WORKFLOW_STARTED, STAGE_STARTED, STAGE_COMPLETED, APPROVAL_REQUESTED, HUMAN_APPROVED, HUMAN_REJECTED, MESSAGE_SENT, MEETING_STARTED, MEETING_ENDED, BUG_CREATED, BUILD_STARTED, PREVIEW_READY, EMERGENCY_STOP, PROVIDER_CONNECTED, PROVIDER_DISCONNECTED. | M | MVP | [Owner] (AGENT_CREATED…HUMAN_REJECTED); danh sách đầy đủ [Đề xuất] |
| FR-118 | Event là nguồn duy nhất cho UI và Virtual Office; không thành phần nào cập nhật UI ngoài event. | M | MVP | [Owner] |
| FR-119 | Template mặc định "FULL SOFTWARE DEVELOPMENT LIFECYCLE" gồm 8 giai đoạn theo thứ tự. | M | MVP | [Owner] |
| FR-120 | Giai đoạn 1 Requirement Intake: nhận prompt, tạo project, PM agent phân tích sơ bộ và thành lập team. | M | MVP | [Owner] |
| FR-121 | Giai đoạn 2 Business Analysis: BA agent tạo tài liệu yêu cầu; có approval gate. | M | MVP | [Owner] |
| FR-122 | Giai đoạn 3 System Design: Architect agent tạo thiết kế hệ thống. | M | MVP | [Owner] |
| FR-123 | Giai đoạn 4 UI/UX Design: Designer agent tạo thiết kế; có approval gate. | M | MVP | [Owner] |
| FR-124 | Giai đoạn 5 Development: FE/BE/DB chạy song song, mỗi nhánh trên git branch riêng, có code review trước khi merge. | M | MVP | [Owner] |
| FR-125 | Giai đoạn 6 QA & Security & Code Review & Product Evaluation: tạo bug record cho mỗi lỗi; bug → task sửa lỗi. | M | MVP | [Owner] |
| FR-126 | Giai đoạn 7 Build & Deployment preview: build và chạy preview; KHÔNG tự động deploy production. | M | MVP | [Owner] |
| FR-127 | Giai đoạn 8 Human Acceptance: Owner approve / reject / feedback / screenshot markup. | M | MVP | [Owner] |
| FR-128 | Khi reject, hệ thống tạo revision task có mục tiêu (targeted) từ feedback, không chạy lại toàn bộ workflow. | M | MVP | [Owner] |
| FR-129 | Mỗi vòng revision tăng số phiên bản của deliverable; lịch sử phiên bản xem được. | M | MVP | [Owner] |
| FR-130 | Screenshot markup: Owner vẽ/ghi chú trên ảnh preview; markup gắn vào feedback và revision task. | S | MVP | [Owner] |
| FR-131 | Tạo project từ prompt ngôn ngữ tự nhiên; project có tên, mô tả, repo, workflow template. | M | MVP | [Owner] |
| FR-132 | Thành lập team: PM agent chọn các vai trò cần thiết từ 18 vai trò; Owner có thể sửa team trước khi chạy. | M | MVP | [Owner]; Owner sửa [Đề xuất] |
| FR-133 | Workflow có thể pause/resume/cancel ở cấp workflow. | M | MVP | [Giả định] |
| FR-134 | Timeline/lịch sử workflow: các giai đoạn, thời gian, agent, kết quả, approval. | S | MVP | [Giả định] |
| FR-135 | Khi một giai đoạn có approval gate bị reject, giai đoạn đó tạo revision task và quay lại gate sau khi sửa. | M | MVP | [Owner] |

### 3.6 Virtual Office 2D (FR-151 … FR-180)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-151 | Văn phòng có 14 phòng: Reception, Management, BA, Architecture, Design Studio, FE, BE, DB, QA, Security, DevOps, Meeting, Relax, Coffee. | M | MVP | [Owner] |
| FR-152 | Zoom và pan bản đồ văn phòng bằng chuột/phím. | M | MVP | [Owner] |
| FR-153 | Nhiều tầng (floors) và chuyển tầng. | C | P2 | [Owner] |
| FR-154 | Chế độ edit layout: di chuyển/đổi kích thước phòng. | C | P2 | [Owner] |
| FR-155 | Đặt/di chuyển furniture trong phòng. | C | P2 | [Owner] |
| FR-156 | Mỗi agent có sprite, outfit (theo phòng ban/vai trò) hiển thị dạng chibi pixel-art. | M | MVP | [Owner] |
| FR-157 | Mỗi agent có bàn (desk) và vị trí mặc định trong phòng của phòng ban. | M | MVP | [Owner] |
| FR-158 | 13 animation: Idle, Walk, Run, Sit, Work, Typing, Reading, Thinking, Meeting, Talking, Celebrating, Error, Waiting. | M | MVP | [Owner] |
| FR-159 | Speech bubble hiển thị tóm tắt hoạt động/tin nhắn gần nhất của agent. | M | MVP | [Owner] |
| FR-160 | Di chuyển và animation của nhân vật chỉ được kích hoạt bởi event thật; không có animation ngẫu nhiên/giả lập. | M | MVP | [Owner] |
| FR-161 | Click vào nhân vật mở popover: trạng thái, task hiện tại, nút Chat/Assign/Pause/Resume/Open Profile. | M | MVP | [Owner] |
| FR-162 | Bảng ánh xạ event → animation/vị trí có thể cấu hình (ví dụ TASK_STARTED → Walk tới desk → Typing). | S | MVP | [Đề xuất] |
| FR-163 | Phòng ban ↔ phòng ánh xạ mặc định; phòng ban tùy chỉnh được gán vào phòng hiện có hoặc phòng mới (P2). | M | MVP | [Giả định] |
| FR-164 | Phòng Relax/Coffee: agent Idle/Available có thể ở đó chỉ khi có event tương ứng (ví dụ AGENT_STATE_CHANGED → Idle). | S | MVP | [Owner] nguyên tắc event-driven |
| FR-165 | Click vào phòng hiển thị danh sách agent và task trong phòng. | S | MVP | [Đề xuất] |

### 3.7 Dashboards (FR-181 … FR-195)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-181 | Overview dashboard: số agent theo trạng thái, project đang chạy, task theo status, approval chờ, chi phí hôm nay/tháng, sự cố. | M | MVP | [Owner]; chỉ số [Đề xuất] |
| FR-182 | Project dashboard: tiến độ theo giai đoạn, task, bug, chi phí, thời gian. | M | MVP | [Owner] |
| FR-183 | Agent dashboard: hiệu suất, token, cost, lỗi, thời gian làm việc. | M | MVP | [Owner] |
| FR-184 | Workflow dashboard: workflow đang chạy, giai đoạn hiện tại, thời gian chờ approval. | S | MVP | [Owner] |
| FR-185 | Cost dashboard: chi phí theo project/agent/phòng ban/provider theo thời gian. | M | MVP | [Owner] |
| FR-186 | Quality dashboard: bug, test pass rate, security findings, reject rate. | S | MVP | [Owner] |
| FR-187 | System dashboard: trạng thái provider, sandbox, hàng đợi, concurrency đang dùng. | S | MVP | [Owner] |
| FR-188 | Mọi dashboard chỉ dùng dữ liệu thật; không có số liệu mẫu/placeholder trong bản release. | M | MVP | [Owner] |

### 3.8 Approval Center (FR-196 … FR-205)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-196 | Approval Center liệt kê mọi yêu cầu phê duyệt: loại, project, giai đoạn, agent yêu cầu, thời gian chờ, artifact đính kèm. | M | MVP | [Owner] |
| FR-197 | Owner approve / reject kèm feedback bằng văn bản; reject bắt buộc có feedback. | M | MVP | [Owner]; bắt buộc feedback [Đề xuất] |
| FR-198 | Approval policy cấu hình được: giai đoạn nào cần approval, loại hành động nào cần approval (deploy, xóa file, chạy lệnh ngoài allow list, vượt ngân sách...). | M | MVP | [Owner] |
| FR-199 | Policy mặc định: BA (stage 2), UI/UX (stage 4), Human Acceptance (stage 8) bắt buộc; production deploy luôn bắt buộc và không tắt được. | M | MVP | [Owner] |
| FR-200 | Approval timeout/escalation (nhắc nhở sau N giờ). | C | P2 | [Đề xuất] |
| FR-201 | Lịch sử phê duyệt: ai, khi nào, quyết định, feedback. | S | MVP | [Giả định] |

### 3.9 Security & Sandbox (FR-206 … FR-230)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-206 | Mỗi project có workspace riêng (thư mục + repo) cô lập với project khác. | M | MVP | [Owner] |
| FR-207 | Thực thi code/lệnh của agent diễn ra trong Docker sandbox. | M | MVP | [Owner]; xem OI-01 |
| FR-208 | Giới hạn tài nguyên sandbox: CPU, RAM, disk, thời gian chạy, số process. | M | MVP | [Owner] |
| FR-209 | Allow/deny list cho lệnh và tool; lệnh ngoài allow list bị chặn hoặc yêu cầu approval theo policy. | M | MVP | [Owner] |
| FR-210 | Hạn chế file system: agent chỉ truy cập workspace của project/task; không truy cập ngoài. | M | MVP | [Owner] |
| FR-211 | Hạn chế network: mặc định chặn, chỉ mở theo allow list domain/port. | M | MVP | [Owner]; mặc định chặn [Đề xuất] |
| FR-212 | Credential isolation: sandbox không chứa credential của provider hay của người dùng. | M | MVP | [Owner] |
| FR-213 | Audit log: mọi lệnh, thay đổi file, truy cập network, thay đổi permission, approval, emergency stop được ghi với timestamp, agent, task. | M | MVP | [Owner] |
| FR-214 | Emergency stop: một nút dừng toàn bộ agent, hủy phiên provider, dừng container, chuyển task RUNNING → BLOCKED, ghi event EMERGENCY_STOP. | M | MVP | [Owner] |
| FR-215 | Agent có permission level (xem `USER_ROLES_AND_PERMISSIONS.md`) quyết định tool/lệnh/FS/network được phép. | M | MVP | [Owner] |
| FR-216 | Khi Docker không khả dụng: hệ thống cảnh báo và hành vi fallback theo quyết định của Owner (chặn hoàn toàn / host-execution giới hạn). | S | MVP | [Chưa chốt — OI-01] |
| FR-217 | Xuất audit log (JSON/CSV). | C | P2 | [Đề xuất] |
| FR-218 | Kill switch per agent/per project (dừng cục bộ) bên cạnh emergency stop toàn cục. | S | MVP | [Đề xuất] |

### 3.10 Multi-agent Orchestration (FR-231 … FR-260)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-231 | Giới hạn số agent chạy đồng thời toàn cục mặc định = 5. | M | MVP | [Owner] |
| FR-232 | Giới hạn này cấu hình được trong Settings, tối đa 18. | M | MVP | [Owner] |
| FR-233 | 18 vai trò mặc định: PM, BA, Architect, UI/UX, FE, BE, DB&Infra, QA, Security, Code Reviewer/Product Evaluator, R&D, Integration, Performance, DevSecOps, Documentation, UX Research, Test Automation, Release Manager. | M | MVP | [Owner] |
| FR-234 | Mỗi agent làm việc trong workspace/git worktree riêng của project. | M | MVP | [Owner] |
| FR-235 | File ownership per task: tập file một task được phép sửa; xung đột ownership bị phát hiện và báo. | M | MVP | [Owner] |
| FR-236 | Message bus nội bộ cho giao tiếp agent–agent và agent–hệ thống; message có cấu trúc và được persist. | M | MVP | [Owner] |
| FR-237 | Resource manager: cấp phát slot concurrency, theo dõi token/thời gian, điều phối hàng đợi. | M | MVP | [Owner] |
| FR-238 | Monitoring dashboard thời gian thực: agent đang chạy, task, tiến độ, log stream. | M | MVP | [Owner] |
| FR-239 | Điều khiển trên monitoring: pause, resume, stop, retry, reassign. | M | MVP | [Owner] |
| FR-240 | Xem logs của agent/task. | M | MVP | [Owner] |
| FR-241 | Xem changes (diff) do agent tạo. | M | MVP | [Owner] |
| FR-242 | Mở workspace của agent (mở thư mục/IDE). | S | MVP | [Owner] |
| FR-243 | Request review: gửi task sang REVIEWING cho reviewer agent. | S | MVP | [Owner] |
| FR-244 | Khi đạt giới hạn concurrency, task READY xếp hàng theo priority. | M | MVP | [Đề xuất] |
| FR-245 | Khi provider trả rate limit, resource manager giảm concurrency tạm thời và retry với backoff. | S | MVP | [Đề xuất — OI-02] |

### 3.11 Premium UI — 22 màn hình (FR-261 … FR-285)

| ID | Màn hình | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-261 | Welcome / Onboarding | M | MVP | [Owner] |
| FR-262 | Main Dashboard | M | MVP | [Owner] |
| FR-263 | Virtual Office | M | MVP | [Owner] |
| FR-264 | Org Chart | S | MVP | [Owner] |
| FR-265 | Department Management | M | MVP | [Owner] |
| FR-266 | Agent Management | M | MVP | [Owner] |
| FR-267 | Agent Detail (10 tab) | M | MVP | [Owner] |
| FR-268 | Agent Builder | M | MVP | [Owner] |
| FR-269 | Skill Library | M | MVP | [Owner] |
| FR-270 | Skill Editor | M | MVP | [Owner] |
| FR-271 | Provider Hub | M | MVP | [Owner] |
| FR-272 | Project Management | M | MVP | [Owner] |
| FR-273 | Project Detail | M | MVP | [Owner] |
| FR-274 | Workflow Builder | S | P2 | [Owner] |
| FR-275 | Task Board | M | MVP | [Owner] |
| FR-276 | Activity Center | M | MVP | [Owner] |
| FR-277 | Approval Center | M | MVP | [Owner] |
| FR-278 | Code Workspace | S | MVP | [Owner] |
| FR-279 | Product Preview | M | MVP | [Owner] |
| FR-280 | QA & Security Report | M | MVP | [Owner] |
| FR-281 | Cost Management | S | MVP | [Owner] |
| FR-282 | Settings | M | MVP | [Owner] |
| FR-283 | Giao diện premium thống nhất (design system, dark/light, animation tinh tế). | S | P2 | [Owner] "premium" |
| FR-284 | Đa ngôn ngữ (Việt/Anh). | C | P2 | [Chưa chốt — OI-07] |

### 3.12 Reliability (FR-286 … FR-300)

| ID | Yêu cầu | Pri | Phase | Nguồn |
|---|---|---|---|---|
| FR-286 | Toàn bộ state (agent, task, workflow, event, message, approval) được lưu bền; không có state chỉ nằm trong bộ nhớ. | M | MVP | [Owner] |
| FR-287 | Sau khi app khởi động lại (hoặc crash), hệ thống khôi phục workflow/task về trạng thái checkpoint gần nhất và tiếp tục hoặc chờ Owner xác nhận. | M | MVP | [Owner] |
| FR-288 | Chính sách retry cấu hình được per agent/task (max retry, backoff). | M | MVP | [Owner] |
| FR-289 | Checkpoint ghi sau mỗi node, mỗi provider turn hoàn tất, và trước mỗi approval gate. | M | MVP | [Owner]; thời điểm [Đề xuất] |
| FR-290 | Phát hiện tiến trình agent/provider chết (heartbeat) và chuyển task sang FAILED hoặc retry. | M | MVP | [Giả định] |
| FR-291 | Thực thi lại một step phải idempotent hoặc có cơ chế bù (compensation) để không tạo trùng artifact. | S | MVP | [Đề xuất] |
| FR-292 | Sao lưu/xuất dữ liệu workspace. | C | P2 | [Đề xuất] |

## 4. Thống kê

- Tổng số FR: **192**.
- Must: 131 · Should: 43 · Could: 18.
- MVP: 167 · P2: 24 · Chưa chốt phase: 1 (FR-098).

## 5. Dependencies

- `SRS.md` (ngữ cảnh, interface), `NON_FUNCTIONAL_REQUIREMENTS.md`, `USER_ROLES_AND_PERMISSIONS.md` (permission level ở FR-215), `ACCEPTANCE_CRITERIA.md` (truy vết).
- Quyết định kiến trúc về sandbox (FR-207, FR-216), shell app (FR-088), event bus (FR-116).

## 6. Rủi ro

- FR-081: CLI provider có thể không expose đủ 13 operation; cần chiến lược NOT_SUPPORTED và ước lượng (FR-096).
- FR-109: PM scoring thiếu dữ liệu đầu vào (workload, provider availability) nếu resource manager chưa hoàn chỉnh.
- FR-160: yêu cầu "chỉ event thật" làm văn phòng "tĩnh" khi không có việc — cần chấp nhận về mặt UX.
- FR-207/FR-080: xung đột host–sandbox.

## 7. Vấn đề chưa giải quyết / mâu thuẫn

| ID | Vấn đề | FR liên quan |
|---|---|---|
| OI-FR-01 | Host execution vs Docker sandbox | FR-080, FR-207, FR-212, FR-216 |
| OI-FR-02 | 18 agent vs rate limit | FR-232, FR-245 |
| OI-FR-03 | Tauri vs web | FR-088, FR-242, FR-278 |
| OI-FR-04 | API key fallback | FR-098 |
| OI-FR-05 | Idle vs Available | FR-024 |
| OI-FR-06 | Meeting là cơ chế thật hay animation | FR-038, FR-158 |
| OI-FR-07 | Đơn vị ngân sách | FR-009, FR-185 |
| OI-FR-08 | Merge conflict ở stage 5 | FR-124 |
| OI-FR-09 | Preview chạy ở đâu | FR-126, FR-279 |
| OI-FR-10 | Multi-company trong MVP | FR-002 |

## 8. Cần người dùng phê duyệt

| # | Hạng mục |
|---|---|
| AP-FR-01 | Toàn bộ priority/phase, đặc biệt các mục đánh dấu [Đề xuất] |
| AP-FR-02 | Danh sách trường task FR-107 |
| AP-FR-03 | Danh mục event FR-117 |
| AP-FR-04 | Chính sách approval mặc định FR-199 |
| AP-FR-05 | Network mặc định chặn FR-211 |
| AP-FR-06 | Fallback khi không có Docker FR-216 |
| AP-FR-07 | Giải quyết OI-FR-01 … OI-FR-10 |
