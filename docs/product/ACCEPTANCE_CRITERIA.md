# ACCEPTANCE CRITERIA (MVP) — AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0) · **Tác giả:** Agent 12 — BA · **Trạng thái:** Draft · **Ngày:** 2026-10-30

> Quy ước: Mỗi AC viết dạng Given/When/Then, có ID AC-xxx, truy vết FR (xem `FUNCTIONAL_REQUIREMENTS.md`) và NFR. Nhóm A–G theo tiêu chí MVP của Owner. [Owner] = tiêu chí gốc; [Đề xuất] = BA cụ thể hóa để kiểm thử được (giá trị số cần Owner chốt).
> Dải mã: A 001–020 · B 021–040 · C 041–070 · D 071–085 · E 086–100 · F 101–120 · G 121–135.

## 1. Mục đích
Chuyển tiêu chí nghiệm thu MVP nhóm A–G thành các mục kiểm thử được, làm cơ sở cho QA viết test plan và cho Owner nghiệm thu MVP.

## 2. Phạm vi
Chỉ MVP (Phase 1). Các FR phase P2 không có AC ở đây.

## 3. Điều kiện chung (Global Preconditions)
- GP-1: Máy Windows 11, Docker Desktop chạy, Git cài, Claude Code CLI và Codex CLI đã cài và login.
- GP-2: App ở trạng thái sạch (workspace mới) trừ khi AC nói khác.
- GP-3: Bản build release (không dữ liệu mock).

---

## Nhóm A — Agent (AC-001 … AC-020)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-001 | Provider Claude Code ở trạng thái Connected | Owner tạo agent qua Agent Builder với đầy đủ trường bắt buộc | Agent được lưu, hiển thị trong Agent Management, trạng thái chuyển Created → Idle → Available qua event AGENT_CREATED / AGENT_STATE_CHANGED | FR-021, FR-024, FR-025 |
| AC-002 | Form Agent Builder | Owner để trống name hoặc nhập name trùng | Hệ thống từ chối lưu và hiển thị lỗi tại trường | FR-022 |
| AC-003 | Form Agent Builder | Owner mở dropdown model | Chỉ model từ ListModels của provider đã chọn được hiển thị | FR-022, FR-093 |
| AC-004 | Agent Available | Task được gán và bắt đầu | Agent lần lượt Assigned → Running; trạng thái giống nhau trên Agent Management, Agent Detail, Task Board, Virtual Office trong ≤ 500 ms (Đề xuất) | FR-025, FR-044, NFR-003 |
| AC-005 | Agent Running | Owner nhấn Pause | Phiên provider bị dừng/hủy, agent → Paused, event AGENT_STATE_CHANGED, nhân vật chuyển Waiting | FR-029 |
| AC-006 | Agent Paused | Owner nhấn Resume | Agent tiếp tục từ checkpoint gần nhất, không chạy lại bước đã hoàn tất | FR-029, FR-102 |
| AC-007 | Agent Running có task | Owner nhấn Disable | Agent → Disabled, task được handover sang agent khác đủ điều kiện kèm log (event TASK_HANDOVER) | FR-030, FR-111 |
| AC-008 | Agent tồn tại | Owner mở Agent Detail | Hiển thị đủ 10 tab: Overview, Profile, Skills, Knowledge, Provider, Tasks, Conversations, Activity Logs, Performance, Permissions | FR-026 |
| AC-009 | Agent Available | Owner chat từ profile | Tin nhắn gửi qua adapter, phản hồi streaming hiển thị, hội thoại lưu trong tab Conversations sau restart | FR-027, FR-040 |
| AC-010 | Hai agent A, B trong cùng project | A gửi message cho B | B nhận được qua message bus; message hiển thị ở Conversations của cả hai; tồn tại sau restart | FR-032, FR-036, FR-040 |
| AC-011 | Agent A có task với file ownership {src/a/**} | A cố ghi file src/b/x.ts thuộc task của B | Thao tác bị chặn, audit log ghi vi phạm, agent nhận thông báo lỗi | FR-039, FR-235 |
| AC-012 | Agent chạy xong 3 task | Owner mở tab Performance | Token usage, cost, số lỗi, thời gian thực thi hiển thị khớp với tổng GetUsage của các session (hoặc đánh dấu ước lượng) | FR-041, FR-096 |
| AC-013 | Agent Running | Agent tạo subtask | Subtask xuất hiện trên Task Board với parent link | FR-035 |
| AC-014 | Agent có trạng thái RUNNING task | Owner xóa agent | Hệ thống từ chối xóa cho tới khi task không còn RUNNING | FR-031 |
| AC-015 | Company mới | Owner tạo agent từ template vai trò "BA" | Form được điền sẵn job title, department, responsibilities, system prompt, permission level mặc định | FR-043 |

## Nhóm B — Provider (AC-021 … AC-040)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-021 | Claude Code CLI đã login | Owner nhấn Connect | CheckConnection thành công, trạng thái Connected, ListModels trả về ≥ 1 model, event PROVIDER_CONNECTED; không yêu cầu API key | FR-076, FR-080, FR-081, FR-082 |
| AC-022 | Codex CLI đã login | Owner nhấn Connect | Như AC-021 với Codex | FR-077, FR-080 |
| AC-023 | CLI chưa cài | Owner nhấn Connect | Trạng thái Not installed, hướng dẫn cài từng bước hiển thị, nút Kiểm tra lại hoạt động | FR-082, FR-083 |
| AC-024 | CLI cài nhưng chưa login | Owner nhấn Connect | Trạng thái Not authenticated, hướng dẫn login qua terminal; app không có ô nhập mật khẩu | FR-083, FR-084 |
| AC-025 | Provider Connected | Agent gửi message qua SendMessage | StreamEvents phát event chuẩn tới UI theo thời gian thực | FR-081, FR-094 |
| AC-026 | Session đang chạy | Owner CancelTask | Tiến trình CLI dừng trong ≤ 5 s (Đề xuất), task → BLOCKED/CANCELLED theo ngữ cảnh | FR-095, NFR-007 |
| AC-027 | Provider Connected | Owner nhấn Disconnect rồi Connect lại | Session cũ đóng; kết nối lại thành công không cần cấu hình thêm | FR-089 |
| AC-028 | Phiên CLI hết hạn | Agent gọi SendMessage | Lỗi SESSION_EXPIRED được hiển thị kèm hướng dẫn login lại; task → BLOCKED, không FAILED | FR-089, NFR-027 |
| AC-029 | Hệ thống chạy 1 ngày với nhiều task | Kiểm tra DB và log bằng scanner secret | Không có token/API key/OAuth plaintext trong DB và log | FR-086, FR-090, NFR-043, NFR-046 |
| AC-030 | UI đang chạy | Bắt toàn bộ payload IPC/HTTP từ backend tới frontend | Không payload nào chứa credential | FR-087, NFR-044 |
| AC-031 | Code review adapter | Rà soát mã nguồn adapter | Không có mã đọc file credential của CLI; chỉ gọi lệnh CLI | FR-085, NFR-042 |
| AC-032 | Hai agent dùng cùng provider | Kiểm tra sandbox của mỗi agent | Không mount/biến môi trường chứa credential; mọi gọi provider đi qua adapter | FR-091, FR-212, NFR-047 |
| AC-033 | Provider Connected | Owner mở Provider Hub | Dashboard hiển thị trạng thái, model, usage, lỗi gần nhất, số session đang mở — dữ liệu thật | FR-092, FR-188 |
| AC-034 | Adapter nhận operation không hỗ trợ (vd. ResumeSession) | Gọi operation | Trả lỗi NOT_SUPPORTED rõ ràng; hệ thống có fallback (tạo session mới với context) | FR-081, FR-097 |

## Nhóm C — Workflow (AC-041 … AC-070)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-041 | Provider Connected, team mặc định | Owner tạo project từ prompt và chọn template SDLC | Project, workspace, git repo được tạo; WORKFLOW_STARTED; Stage 1 chạy | FR-119, FR-120, FR-131, FR-206 |
| AC-042 | Stage 1 chạy | PM agent hoàn thành phân tích | Team đề xuất hiển thị để Owner xem/sửa; backlog sơ bộ được tạo | FR-120, FR-132 |
| AC-043 | Stage 2 BA hoàn thành | — | Task → WAITING_APPROVAL, APPROVAL_REQUESTED xuất hiện ở Approval Center; workflow không tiến sang Stage 3 | FR-106, FR-121, FR-196 |
| AC-044 | Yêu cầu approval Stage 2 | Owner Approve | HUMAN_APPROVED; Stage 3 bắt đầu | FR-197 |
| AC-045 | Yêu cầu approval Stage 2 | Owner Reject với feedback | HUMAN_REJECTED; revision task được tạo cho BA agent với feedback; version +1; sau khi sửa quay lại gate | FR-135, FR-129 |
| AC-046 | Yêu cầu approval | Owner Reject không nhập feedback | Hệ thống không cho gửi (Đề xuất) | FR-197 |
| AC-047 | Stage 4 UI/UX hoàn thành | — | Approval gate như AC-043 | FR-123 |
| AC-048 | Stage 5 bắt đầu | — | Ít nhất 3 task FE/BE/DB ở trạng thái RUNNING song song (nếu concurrency ≥ 3), mỗi task có branch/worktree riêng | FR-105, FR-124, FR-234 |
| AC-049 | Task dev hoàn thành code | — | Task → REVIEWING; Code Reviewer agent review; chỉ merge sau khi review pass | FR-124, FR-243 |
| AC-050 | Stage 6 chạy, QA phát hiện lỗi | — | BUG_CREATED; bug record có link tới task sửa; task sửa được gán | FR-125 |
| AC-051 | Stage 7 hoàn thành | — | PREVIEW_READY; Product Preview mở được; không có bước deploy production nào xảy ra (audit log xác nhận) | FR-126, NFR-056 |
| AC-052 | Stage 8 | Owner Approve | Project COMPLETED; HUMAN_APPROVED; nhân vật Celebrating (nếu mapping) | FR-127 |
| AC-053 | Stage 8 | Owner Reject với 2 feedback + 1 screenshot markup | 2 revision task targeted được tạo (không tạo lại task Stage 1–4); markup gắn vào task; deliverable version n+1 | FR-128, FR-129, FR-130 |
| AC-054 | Revision hoàn tất | — | Preview version mới; lịch sử version hiển thị cả n và n+1 | FR-129 |
| AC-055 | Task READY, không assignee | AUTO assignment chạy | Agent được chọn và lý do (điểm theo role, skills, tools, workload, provider, permissions) được ghi lại, xem được | FR-109 |
| AC-056 | Task READY | Owner assign thủ công | Agent được gán, mode MANUAL, ghi đè AUTO | FR-110 |
| AC-057 | Task RUNNING, agent thất bại hết max retry | — | TASK_HANDOVER sang agent khác đủ điều kiện kèm context/log; nếu không có agent → task FAILED + thông báo | FR-103, FR-111 |
| AC-058 | Workflow chạy | Kiểm tra event store | Mọi event có id, type, timestamp, actor, entity, payload, correlation id | FR-116 |
| AC-059 | Workflow chạy | Tắt event bus tới UI (test) | UI không cập nhật (chứng minh UI chỉ phụ thuộc event) | FR-118 |
| AC-060 | Task tồn tại | Đi qua toàn bộ vòng đời | Chỉ xuất hiện 10 trạng thái hợp lệ; chuyển tiếp không hợp lệ bị từ chối | FR-108 |
| AC-061 | Workflow có template lưu | Owner tạo project mới chọn template đó | Workflow chạy đúng cấu trúc template | FR-114 |
| AC-062 | Workflow RUNNING | Owner Pause workflow rồi Resume | Mọi task tạm dừng và tiếp tục từ checkpoint | FR-133 |
| AC-063 | Concurrency = 5, 8 task READY | Workflow chạy | Tối đa 5 RUNNING; 3 còn lại xếp hàng theo priority; UI responsive | FR-231, FR-244, NFR-012 |
| AC-064 | Settings concurrency = 18 | 18 task READY với 18 agent | 18 slot được cấp (thông lượng thực phụ thuộc provider; nếu RATE_LIMITED → backoff, không FAILED) | FR-232, FR-245, NFR-001 |

## Nhóm D — Skill (AC-071 … AC-085)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-071 | App mới | Owner mở Skill Library | 6 category (Development, Architecture, Database, Testing, Security, DevOps) với ≥ 1 skill ví dụ mỗi category | FR-051 |
| AC-072 | Skill Library | Owner tạo custom skill với đủ trường, upload 1 MD và 1 PDF | Thư mục `skills/<name>/` có SKILL.md, references/ chứa 2 file, examples/, scripts/ | FR-053, FR-054, FR-055 |
| AC-073 | Skill v1 tồn tại | Owner sửa và lưu version mới | v2 được tạo; agent gán v1 không đổi cho tới khi cập nhật | FR-056 |
| AC-074 | Agent tồn tại | Owner nhập chỉ dẫn dạy agent | Danh sách rule có cấu trúc hiển thị; gợi ý tạo/cập nhật skill; chưa có gì được lưu | FR-058, FR-059, FR-060, FR-061 |
| AC-075 | Danh sách rule gợi ý | Owner xóa 1 rule, sửa 1 rule rồi Lưu | Skill lưu đúng nội dung sau chỉnh sửa | FR-061 |
| AC-076 | Owner đóng màn hình Teach Agent trước khi Lưu | — | Không skill/rule nào được persist | FR-061 |
| AC-077 | Skill đã lưu | Owner gán cho 3 agent | 3 agent có skill trong tab Skills; skill hiển thị 3 agent đang dùng | FR-062 |
| AC-078 | Skill gán cho agent | Hệ thống verify | Agent tóm tắt lại rule; kết quả verify (pass/fail) hiển thị; chỉ pass mới đánh dấu "đã load" | FR-063 |
| AC-079 | Skill có test case | Owner chạy evaluation | Kết quả pass/fail từng test hiển thị | FR-064 |
| AC-080 | Agent có 10 skill, task thuộc Testing | Task bắt đầu | Activity Logs ghi chỉ skill liên quan (Testing + mandatory chung) được nạp, không nạp cả 10 | FR-065 |
| AC-081 | Skill có mandatory rule | Agent chạy task | Prompt gửi provider chứa mandatory rule được đánh dấu bắt buộc (kiểm tra qua log có mask) | FR-066 |
| AC-082 | Tab Knowledge | Owner gỡ skill | Agent không còn nạp skill đó ở task tiếp theo | FR-068 |

## Nhóm E — Virtual Office (AC-086 … AC-100)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-086 | App mở | Owner mở Virtual Office | 14 phòng hiển thị đúng tên: Reception, Management, BA, Architecture, Design Studio, FE, BE, DB, QA, Security, DevOps, Meeting, Relax, Coffee | FR-151 |
| AC-087 | Virtual Office | Owner zoom/pan | Bản đồ phóng to/thu nhỏ và kéo mượt (≥ 30 FPS Đề xuất) | FR-152, NFR-004 |
| AC-088 | Agent được tạo trong phòng ban Frontend | AGENT_CREATED | Nhân vật chibi xuất hiện và di chuyển tới desk trong phòng FE | FR-156, FR-157, FR-163 |
| AC-089 | Agent Available | TASK_STARTED | Nhân vật chuyển animation Typing/Work tại desk; speech bubble hiển thị tên task | FR-158, FR-159, FR-160 |
| AC-090 | Agent Running | MEETING_STARTED với agent khác | Các nhân vật Walk tới phòng Meeting và animation Meeting/Talking | FR-158, FR-160 |
| AC-091 | Không có event nào trong 5 phút | Quan sát | Không nhân vật nào di chuyển/đổi animation (ngoài Idle loop tại chỗ) | FR-160 |
| AC-092 | Agent gặp lỗi (TASK_FAILED) | — | Animation Error; speech bubble lỗi | FR-158 |
| AC-093 | Task WAITING_APPROVAL | — | Animation Waiting | FR-158 |
| AC-094 | Nhân vật trên màn hình | Owner click | Popover với trạng thái, task hiện tại, nút Chat/Assign/Pause/Resume/Profile hoạt động | FR-161 |
| AC-095 | 18 agent Running đồng thời | Quan sát | Tất cả 18 nhân vật hiển thị đúng trạng thái; FPS ≥ 30 (Đề xuất) trên máy tham chiếu | FR-158, NFR-004 |
| AC-096 | Virtual Office | Kiểm tra toàn bộ 13 animation qua event mô phỏng (test harness, không phải production) | Mỗi animation Idle, Walk, Run, Sit, Work, Typing, Reading, Thinking, Meeting, Talking, Celebrating, Error, Waiting đều render được | FR-158 |

## Nhóm F — Security (AC-101 … AC-120)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-101 | Task có sandbox | Agent chạy lệnh | Lệnh thực thi trong Docker container với mount workspace task; không chạy trên host (xác minh qua audit + docker ps) | FR-207 |
| AC-102 | Sandbox với limit CPU/RAM/time | Agent chạy lệnh vượt giới hạn | Tiến trình bị dừng; task → FAILED/BLOCKED; audit ghi | FR-208, NFR-050 |
| AC-103 | Deny list chứa `rm -rf /` | Agent cố chạy | Bị chặn; audit log ghi; agent nhận thông báo | FR-209, NFR-051 |
| AC-104 | Lệnh không thuộc allow lẫn deny, POL-06 bật | Agent chạy | Yêu cầu approval xuất hiện; lệnh không chạy cho tới khi approve | FR-209, FR-198 |
| AC-105 | Sandbox FS giới hạn | Agent cố đọc file ngoài workspace (vd. thư mục home) | Thất bại; audit ghi | FR-210, NFR-048 |
| AC-106 | Network allow list rỗng (mặc định deny Đề xuất) | Agent cố curl ra ngoài | Thất bại; audit ghi | FR-211, NFR-049 |
| AC-107 | Hai project | Agent project A cố truy cập workspace project B | Thất bại | FR-206 |
| AC-108 | Hệ thống chạy | Owner xem audit log | Mọi lệnh, thay đổi file, approval, emergency stop có timestamp, agent, task; không sửa/xóa được từ UI | FR-213, NFR-052 |
| AC-109 | 5 agent Running | Owner nhấn Emergency Stop | Trong ≤ 5 s (Đề xuất): mọi session cancel, container dừng, task → BLOCKED, agent → Paused, EMERGENCY_STOP ghi; banner hiển thị | FR-214, NFR-007, NFR-053 |
| AC-110 | Sau Emergency Stop | Owner Resume | Task tiếp tục từ checkpoint | FR-214, FR-102 |
| AC-111 | Agent permission L1 | Agent cố tạo branch/commit | Bị chặn theo permission level; audit ghi | FR-215, NFR-054 |
| AC-112 | Mọi màn hình | Kiểm tra UI | Nút Emergency Stop hiển thị và hoạt động trên mọi màn hình | NFR-053 |
| AC-113 | Toàn bộ test F chạy xong | Scan DB/log/sandbox | 0 credential plaintext; 0 credential trong container | FR-086, FR-090, FR-212 |
| AC-114 | Docker không chạy | Owner tạo project | Cảnh báo rõ ràng; hành vi theo quyết định Owner (FR-216 — chưa chốt) | FR-216 |

## Nhóm G — Reliability (AC-121 … AC-135)

| ID | Given | When | Then | FR |
|---|---|---|---|---|
| AC-121 | Workflow ở Stage 5 với 3 task RUNNING | Kill app (task kill) rồi mở lại | Dashboard hiển thị đúng project/task/agent như trước; không mất event; banner "N workflow đang chạy" | FR-286, FR-287, NFR-021 |
| AC-122 | Sau AC-121 | Owner chọn Tiếp tục | Task resume từ checkpoint; bước đã hoàn tất không chạy lại; artifact không bị tạo trùng | FR-287, FR-289, FR-291 |
| AC-123 | Task thất bại tạm thời (vd. provider timeout) | — | Retry tự động theo max retry/backoff; retry count hiển thị | FR-103, FR-288, NFR-023 |
| AC-124 | Tiến trình CLI bị kill từ bên ngoài | — | Heartbeat phát hiện trong ≤ 60 s (Đề xuất); task → FAILED hoặc retry | FR-290, NFR-024 |
| AC-125 | Approval WAITING khi app tắt | Mở lại app | Approval vẫn chờ ở Approval Center với artifact đầy đủ | FR-286 |
| AC-126 | Hội thoại agent và message bus có dữ liệu | Restart | Conversations và message còn nguyên | FR-040, FR-286 |
| AC-127 | 20 lần chạy kịch bản AC-121 | — | 20/20 khôi phục thành công | NFR-026 |
| AC-128 | Event store 10k event | Khởi động app | Sẵn sàng ≤ 10 s (Đề xuất) | NFR-008 |
| AC-129 | Task vượt max execution time | — | Task dừng, → FAILED/BLOCKED; không treo vô hạn | NFR-029 |
| AC-130 | Provider mất kết nối giữa task | — | Task → BLOCKED; khi provider trở lại, tự nối và tiếp tục | NFR-027 |

## 4. Ma trận truy vết tóm tắt

| Nhóm | Số AC | FR chính |
|---|---|---|
| A Agent | 15 | FR-021…044 |
| B Provider | 14 | FR-076…097 |
| C Workflow | 24 | FR-101…135, FR-231…245 |
| D Skill | 12 | FR-051…068 |
| E Virtual Office | 11 | FR-151…165 |
| F Security | 14 | FR-206…218 |
| G Reliability | 10 | FR-286…291 |
| **Tổng** | **100** | |

## 5. Dependencies
- Test harness mô phỏng event (AC-096) phải tách khỏi production để không vi phạm FR-160.
- Máy tham chiếu cho AC-087, AC-095, AC-128.
- Scanner secret cho AC-029, AC-113.

## 6. Rủi ro
- AC-101 không đạt được nếu kiến trúc chọn chạy CLI trên host (OI host vs sandbox) → phải viết lại AC-101/105/106 theo cơ chế thực thi được chọn.
- AC-064 chỉ kiểm tra "slot", không bảo đảm thông lượng thật.
- AC-034 phụ thuộc khả năng của từng CLI.

## 7. Vấn đề chưa giải quyết / mâu thuẫn
| ID | Vấn đề |
|---|---|
| OI-AC-01 | Host vs sandbox quyết định cách kiểm thử nhóm F. |
| OI-AC-02 | 18 agent vs rate limit: tiêu chí AC-064 chấp nhận được không? |
| OI-AC-03 | Tauri vs web ảnh hưởng AC-030 (cách bắt payload). |
| OI-AC-04 | Các giá trị số (500 ms, 5 s, 60 s, 10 s, 30 FPS) là Đề xuất. |
| OI-AC-05 | AC-046 (bắt buộc feedback) là Đề xuất. |
| OI-AC-06 | AC-114 phụ thuộc quyết định FR-216. |

## 8. Cần người dùng phê duyệt
| # | Hạng mục |
|---|---|
| AP-AC-01 | Toàn bộ 100 AC, đặc biệt các giá trị Đề xuất |
| AP-AC-02 | Quy ước "pass MVP" = 100% AC Must-FR pass, hay cho phép tỷ lệ (Đề xuất: 100% nhóm B, F, G; ≥ 95% nhóm còn lại) |
| AP-AC-03 | Giải quyết OI-AC-01…06 |
