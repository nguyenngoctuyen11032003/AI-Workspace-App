# USER FLOWS — AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0) · **Tác giả:** Agent 12 — BA · **Trạng thái:** Draft · **Ngày:** 2026-10-30

> **[OWR-3 A.16/A.15]** Tên event trong các flow dưới (UPPER_SNAKE như `APPROVAL_REQUESTED`, `HUMAN_APPROVED`) là tên tương thích/hiển thị; canonical là dotted lowercase (`EVENT_SYSTEM.md` §0, §5.7). Chỉ 16 tên ở §0 là event gốc của Owner; tên khác là đề xuất, chờ Owner. Task ở `WAITING_APPROVAL` không tính vào 5 active workers nếu worker đã được giải phóng và trạng thái đã persist (vẫn theo quota provider).
>
> Quy ước: [Owner] = yêu cầu gốc; [Giả định]/[Đề xuất] = BA. Mỗi flow ghi: Actor, Tiền điều kiện, Các bước (Actor → Hệ thống), Event phát sinh (FR-117), Kết quả, Ngoại lệ, FR liên quan.

## 1. Mục đích
Mô tả từng bước 15 hành trình người dùng chủ chốt để UI/UX, kiến trúc và QA thống nhất hành vi hệ thống trước khi thiết kế màn hình.

## 2. Phạm vi
15 flow chính của MVP (và một số bước P2 được đánh dấu). Không mô tả UI chi tiết (wireframe thuộc `docs/design/`).

## 3. Danh sách flow

| # | Flow | Màn hình chính |
|---|---|---|
| UF-01 | Onboarding & kết nối provider | Welcome/Onboarding, Provider Hub |
| UF-02 | Tạo company & phòng ban | Department Management, Org Chart |
| UF-03 | Tạo agent (Agent Builder) | Agent Builder, Agent Management |
| UF-04 | Dạy agent (Teach Agent) | Agent Detail/Knowledge, Skill Editor |
| UF-05 | Tạo custom skill & gán | Skill Library, Skill Editor |
| UF-06 | Tạo project từ prompt → nghiệm thu (SDLC 8 giai đoạn) | Project Mgmt, Approval Center, Product Preview |
| UF-07 | Phê duyệt tại approval gate (BA / UI/UX) | Approval Center |
| UF-08 | Reject tại Human Acceptance → revision | Product Preview, Approval Center, Task Board |
| UF-09 | Emergency stop & khôi phục | Mọi màn hình, Activity Center |
| UF-10 | Pause / Resume / Disable agent | Agent Detail, Virtual Office |
| UF-11 | Assign task thủ công & reassign | Task Board, Agent Detail |
| UF-12 | Quan sát & tương tác Virtual Office | Virtual Office |
| UF-13 | Theo dõi chi phí & ngân sách | Cost Management, Dashboard |
| UF-14 | Dựng workflow bằng Workflow Builder (P2) | Workflow Builder |
| UF-15 | Khôi phục sau restart/crash | Welcome, Activity Center |

---

## UF-01 — Onboarding & kết nối provider
**Actor:** Owner. **Tiền điều kiện:** App cài xong; Claude Code CLI và/hoặc Codex CLI đã cài (có thể chưa login). **FR:** FR-080…FR-092, FR-261, FR-271.

1. Owner mở app lần đầu → Hệ thống hiển thị Welcome: giới thiệu, chọn ngôn ngữ [chưa chốt], nút "Bắt đầu".
2. Owner nhập tên company → Hệ thống tạo company + 12 phòng ban mặc định (FR-003); event COMPANY_CREATED [Đề xuất bổ sung event].
3. Hệ thống quét provider: chạy CheckConnection cho Claude Code, Codex → hiển thị trạng thái: Not installed / Installed, not logged in / Connected.
4. Nếu Not installed → Hệ thống hiển thị hướng dẫn cài (lệnh, link) và nút "Kiểm tra lại".
5. Nếu Not logged in → Hệ thống hướng dẫn chạy lệnh login của CLI trong terminal **bên ngoài** (không nhập mật khẩu trong app — FR-084); nút "Tôi đã đăng nhập" → CheckConnection lại.
6. Khi Connected → Hệ thống gọi ListModels, lưu trạng thái provider; event PROVIDER_CONNECTED.
7. Hệ thống kiểm tra Docker: có → OK; không → cảnh báo và hiển thị lựa chọn theo quyết định Owner (FR-216, chưa chốt).
8. Hệ thống đề nghị tạo đội ngũ mặc định từ 18 vai trò (FR-043) → Owner chọn tất cả hoặc một phần → tạo agent hàng loạt, mỗi agent được gán provider/model mặc định.
9. Hệ thống mở Main Dashboard; Virtual Office hiển thị agent tại bàn của phòng ban (event AGENT_CREATED → nhân vật xuất hiện).

**Ngoại lệ:** CLI phiên bản không hỗ trợ → thông báo + link; Docker thiếu → hạn chế tính năng; không provider nào Connected → vẫn cho tạo agent nhưng agent ở Idle.

## UF-02 — Tạo company & phòng ban
**Actor:** Owner. **FR:** FR-001…FR-011.

1. Owner mở Department Management → danh sách 12 phòng ban.
2. Owner "Thêm phòng ban" → nhập tên (vd. Mobile), mô tả, màu, chọn phòng Virtual Office ánh xạ (hiện có hoặc "Mới" ở P2).
3. Hệ thống lưu; event DEPARTMENT_CREATED [Đề xuất].
4. Owner chỉ định trưởng phòng từ danh sách agent thuộc phòng (FR-005).
5. Owner mở Org Chart → xem cây; (P2) kéo-thả agent sang phòng khác → hệ thống cập nhật department và reporting manager, phát event AGENT_UPDATED → nhân vật đổi bàn trong Virtual Office.

**Ngoại lệ:** Xóa phòng còn agent → chặn, yêu cầu chuyển agent.

## UF-03 — Tạo agent (Agent Builder)
**Actor:** Owner. **Tiền điều kiện:** ≥ 1 provider Connected. **FR:** FR-021…FR-025, FR-043, FR-093, FR-268.

1. Owner mở Agent Management → "Tạo agent" → chọn "Từ template vai trò" hoặc "Trống".
2. Hệ thống điền sẵn (nếu template): job title, department, responsibilities, system prompt, skills gợi ý, permission level mặc định.
3. Owner điền/sửa: name, avatar, provider (chỉ provider Connected), model (từ ListModels), tools, memory config, working directory, max token budget, max execution time, max retry, reporting manager, status.
4. Owner "Lưu" → Hệ thống validate (FR-022) → tạo agent trạng thái Created → event AGENT_CREATED → Idle → nếu provider Connected → Available (AGENT_STATE_CHANGED).
5. Virtual Office: nhân vật xuất hiện tại Reception rồi Walk tới desk trong phòng của phòng ban (theo mapping FR-162).
6. Owner mở Agent Detail → kiểm tra tab Overview/Provider/Permissions.

**Ngoại lệ:** Tên trùng; provider Disconnected → không cho chọn; model không hợp lệ.

## UF-04 — Dạy agent (Teach Agent)
**Actor:** Owner. **Tiền điều kiện:** Agent tồn tại. **FR:** FR-058…FR-068.

1. Owner mở Agent Detail → tab Knowledge → "Dạy agent".
2. Owner nhập chỉ dẫn (text) hoặc upload file MD/PDF (vd. "Luôn viết test trước khi code; dùng conventional commits").
3. Hệ thống parse thành danh sách rule có cấu trúc (FR-059) và hiển thị để review.
4. Hệ thống gợi ý: tạo skill mới "X" (category, tên) hoặc bổ sung vào skill hiện có (FR-060).
5. Owner review: sửa/xóa từng rule, chọn mandatory rules, allowed tools; chọn tạo mới/cập nhật; đặt version (FR-061, FR-056).
6. Owner "Lưu vào library" → Hệ thống ghi `skills/<name>/SKILL.md` (+ references nếu upload) (FR-055).
7. Hệ thống gán skill cho agent (FR-062) → verify: tạo session, yêu cầu agent tóm tắt rule đã nạp → so khớp (FR-063) → hiển thị kết quả.
8. (Tùy chọn) Owner chạy evaluation với test case của skill (FR-064) → pass/fail.
9. Tab Knowledge hiển thị skill + rule; Owner có thể gỡ/xóa bất kỳ lúc nào (FR-068).

**Ngoại lệ:** Parse không ra rule → cho phép soạn tay; verify thất bại → cảnh báo, không đánh dấu "đã học".

## UF-05 — Tạo custom skill & gán
**Actor:** Owner. **FR:** FR-051…FR-057, FR-062, FR-065.

1. Owner mở Skill Library → duyệt theo category → "Tạo skill".
2. Skill Editor: name, description, category, instructions, references (upload), code examples, checklists, mandatory rules, allowed tools, tests, version.
3. Owner "Lưu" → thư mục chuẩn được tạo; event SKILL_CREATED [Đề xuất].
4. Owner "Gán" → chọn nhiều agent (many-to-many).
5. (P2) Owner "Test skill" → chọn agent thử + prompt mẫu → xem kết quả.
6. Khi agent nhận task, hệ thống chỉ nạp skill liên quan (FR-065) → tab Activity Logs ghi "Loaded skills: ...".

## UF-06 — Tạo project từ prompt → nghiệm thu (SDLC 8 giai đoạn)
**Actor:** Owner, các agent. **Tiền điều kiện:** Provider Connected; ≥ các agent vai trò cần thiết; Docker OK. **FR:** FR-101…FR-135, FR-196…FR-199, FR-231…FR-245.

1. Owner mở Project Management → "Dự án mới" → nhập prompt tự nhiên (vd. "Xây dựng web quản lý kho nhỏ với đăng nhập, danh sách sản phẩm, nhập/xuất kho") → chọn template "FULL SOFTWARE DEVELOPMENT LIFECYCLE".
2. Hệ thống tạo project + workspace + git repo (FR-131, FR-206); event PROJECT_CREATED, WORKFLOW_STARTED.
3. **Stage 1 Requirement Intake:** PM agent phân tích prompt, đề xuất team (FR-132) → Owner xem/sửa team → xác nhận; PM tạo backlog sơ bộ; STAGE_COMPLETED.
4. **Stage 2 BA:** BA agent viết tài liệu yêu cầu → task WAITING_APPROVAL → APPROVAL_REQUESTED → Owner duyệt tại Approval Center (UF-07). Approve → Stage 3.
5. **Stage 3 System Design:** Architect agent viết thiết kế (approval tùy policy POL-05).
6. **Stage 4 UI/UX:** Designer agent tạo thiết kế → approval gate (UF-07).
7. **Stage 5 Development:** Hệ thống tạo task FE/BE/DB song song (Parallel node), mỗi task có file ownership và branch/worktree riêng (FR-124, FR-234, FR-235); resource manager cấp slot theo concurrency (FR-231, FR-244). Agent code → commit → task REVIEWING → Code Reviewer agent review → sửa nếu cần → merge nội bộ → Merge node.
8. **Stage 6 QA & Security & Review & Evaluation:** các agent chạy test/scan/evaluate trong sandbox → mỗi lỗi tạo BUG_CREATED → bug → task sửa → dev sửa → QA re-test (vòng lặp giới hạn bởi max retry/policy).
9. **Stage 7 Build & Preview:** DevOps/Release agent build, chạy preview trong sandbox (cổng/địa chỉ do kiến trúc quyết định — OI) → PREVIEW_READY; **không deploy production** (FR-126).
10. **Stage 8 Human Acceptance:** Owner mở Product Preview → dùng thử → xem QA & Security Report → Approve (→ project COMPLETED, event HUMAN_APPROVED) hoặc Reject (UF-08).

Virtual Office trong suốt flow: agent Walk tới phòng tương ứng, Typing khi RUNNING, Meeting khi MEETING_STARTED, Waiting khi WAITING_APPROVAL, Celebrating khi HUMAN_APPROVED [mapping Đề xuất].

**Ngoại lệ:** Agent FAILED hết retry → handover (FR-111); rate limit → queue/backoff (FR-245); vượt ngân sách → POL-08 approval.

## UF-07 — Phê duyệt tại approval gate
**Actor:** Owner. **FR:** FR-106, FR-196…FR-201.

1. Hệ thống phát APPROVAL_REQUESTED → badge trên Approval Center + thông báo.
2. Owner mở Approval Center → chọn yêu cầu → xem artifact (tài liệu BA / thiết kế UI) và ngữ cảnh (project, stage, agent).
3. Owner chọn Approve (kèm ghi chú tùy chọn) hoặc Reject (bắt buộc feedback).
4. Approve → HUMAN_APPROVED → workflow tiếp tục. Reject → HUMAN_REJECTED → hệ thống tạo revision task cho agent của stage đó với feedback (FR-135) → sau khi sửa, quay lại gate (version +1).

## UF-08 — Reject tại Human Acceptance → revision
**Actor:** Owner. **FR:** FR-127…FR-130.

1. Trong Product Preview, Owner phát hiện lỗi → "Reject".
2. Owner viết feedback; chụp screenshot → vẽ/ghi chú markup (FR-130); có thể tạo nhiều mục feedback.
3. Hệ thống phát HUMAN_REJECTED; PM agent phân tích feedback → tạo revision tasks targeted (vd. "Sửa validation form đăng nhập", "Đổi màu nút theo markup") gắn tới task/file gốc (FR-128); deliverable version n → n+1 (FR-129).
4. Chỉ các giai đoạn liên quan (5–7) chạy lại cho phạm vi đó; stage 1–4 không chạy lại.
5. Sau khi sửa, QA re-test phạm vi liên quan → build preview mới → quay lại Stage 8 với version mới.
6. Owner so sánh version (lịch sử revision) → Approve.

**Ngoại lệ:** Feedback mơ hồ → PM tạo task "Làm rõ yêu cầu" và hỏi lại Owner qua chat (WAITING).

## UF-09 — Emergency stop & khôi phục
**Actor:** Owner. **FR:** FR-214, FR-218, FR-095.

1. Owner nhấn nút Emergency Stop (luôn hiển thị) → xác nhận.
2. Hệ thống phát EMERGENCY_STOP → CancelTask mọi session → dừng/hủy container → task RUNNING → BLOCKED, agent → Paused → audit log.
3. UI hiển thị banner "Đã dừng khẩn cấp" + danh sách task bị ảnh hưởng; Virtual Office: mọi nhân vật Error/Waiting.
4. Owner xem logs/changes của từng task; quyết định: Resume tất cả / Resume từng task / Cancel task / Reassign.
5. Resume → agent Running từ checkpoint gần nhất (FR-102); event TASK_STARTED.

**Ngoại lệ:** Tiến trình không dừng trong 5 s → kill cứng, ghi audit.

## UF-10 — Pause / Resume / Disable agent
**Actor:** Owner. **FR:** FR-029…FR-031.

1. Owner mở Agent Detail (hoặc click nhân vật trong Virtual Office) → Pause.
2. Hệ thống tạm dừng phiên provider (CancelTask hoặc giữ, theo chính sách) → agent Paused → task giữ RUNNING/BLOCKED [Đề xuất: BLOCKED] → Virtual Office: Waiting.
3. Resume → ResumeSession (nếu hỗ trợ) hoặc tạo session mới với context checkpoint → Running.
4. Disable → agent không nhận task mới; task đang chạy → handover (FR-111) → nhân vật rời bàn về Reception [Đề xuất].

## UF-11 — Assign task thủ công & reassign
**Actor:** Owner. **FR:** FR-109…FR-111, FR-239.

1. Owner mở Task Board → chọn task READY → "Assign" → danh sách agent kèm điểm PM scoring gợi ý (role, skills, tools, workload, provider, permissions).
2. Owner chọn agent → TASK_ASSIGNED (mode MANUAL) → agent Assigned → chạy khi có slot.
3. Reassign task RUNNING: Owner chọn "Reassign" → hệ thống cancel session hiện tại, chuyển context/log sang agent mới (TASK_HANDOVER).

## UF-12 — Quan sát & tương tác Virtual Office
**Actor:** Owner. **FR:** FR-151…FR-165.

1. Owner mở Virtual Office → thấy 14 phòng; agent tại bàn; speech bubble tóm tắt hoạt động.
2. Zoom/pan; (P2) chuyển tầng, chỉnh layout.
3. Click nhân vật → popover: trạng thái, task, nút Chat/Assign/Pause/Resume/Profile (FR-161).
4. Click phòng → danh sách agent/task trong phòng (FR-165).
5. Mọi chuyển động chỉ khi có event; khi không có việc, agent Idle tại bàn hoặc Relax/Coffee theo event trạng thái (FR-160, FR-164).

## UF-13 — Theo dõi chi phí & ngân sách
**Actor:** Owner. **FR:** FR-041, FR-096, FR-185, FR-009.

1. Owner mở Cost Management → chi phí theo project/agent/phòng ban/provider, theo ngày/tháng; dữ liệu từ GetUsage (đánh dấu "ước lượng" nếu CLI không cung cấp).
2. Owner đặt ngân sách phòng ban/project (P2) → cảnh báo 80%/100% → POL-08 khi vượt.
3. Drill-down tới task/session cụ thể.

## UF-14 — Dựng workflow bằng Workflow Builder (P2)
**Actor:** Owner. **FR:** FR-112…FR-115.

1. Owner mở Workflow Builder → canvas trống hoặc clone template SDLC.
2. Kéo-thả node (Agent, Approval, Condition, Parallel, Merge, Tool, Test, Deployment) → nối cạnh → cấu hình từng node (vai trò, điều kiện, approver...).
3. "Dry-run" → hệ thống mô phỏng đường đi, báo lỗi cấu hình (nhánh không merge, deployment production thiếu approval).
4. "Lưu template" → dùng được khi tạo project.

## UF-15 — Khôi phục sau restart/crash
**Actor:** Owner, Hệ thống. **FR:** FR-286…FR-291.

1. App khởi động → đọc checkpoint/event store → dựng lại read-model → hiển thị Welcome/Dashboard với banner "Phát hiện N workflow đang chạy trước khi tắt".
2. Owner xem danh sách; với mỗi workflow: Tiếp tục / Tạm dừng / Hủy [Đề xuất: mặc định chờ Owner].
3. Tiếp tục → engine resume từ checkpoint; ResumeSession nếu được, ngược lại tạo session mới với context đã lưu; task → RUNNING; Virtual Office cập nhật.
4. Task không thể khôi phục (vd. container mất) → FAILED → retry/handover theo chính sách.

## 4. Dependencies
`FUNCTIONAL_REQUIREMENTS.md`, `USER_ROLES_AND_PERMISSIONS.md`, `docs/design/` (wireframe), `docs/architecture/` (session resume, sandbox, preview hosting).

## 5. Rủi ro
- Flow UF-06 dài; nếu approval gate nhiều, Owner bị gián đoạn liên tục → cần policy linh hoạt.
- UF-09: nếu CLI không hỗ trợ cancel sạch, phải kill process → mất context.
- UF-15: nếu ResumeSession không khả dụng, chi phí token chạy lại.

## 6. Vấn đề chưa giải quyết / mâu thuẫn
| ID | Vấn đề |
|---|---|
| OI-UF-01 | UF-01 bước 5: login CLI phải thực hiện ngoài app — Owner chấp nhận UX này? (hệ quả của luật "không hỏi mật khẩu") |
| OI-UF-02 | UF-01 bước 7: hành vi khi không có Docker (host vs sandbox). |
| OI-UF-03 | UF-06 bước 9: preview chạy ở đâu, mở bằng gì (Product Preview nhúng webview? Tauri vs web). |
| OI-UF-04 | UF-06 bước 7: 18 agent song song vs rate limit → thời gian thực tế. |
| OI-UF-05 | UF-10: Pause giữ task RUNNING hay BLOCKED. |
| OI-UF-06 | UF-15: mặc định tự động tiếp tục hay chờ Owner. |
| OI-UF-07 | Event bổ sung (COMPANY_CREATED, DEPARTMENT_CREATED, SKILL_CREATED, PROJECT_CREATED) chưa có trong danh sách Owner. |

## 7. Cần người dùng phê duyệt
| # | Hạng mục |
|---|---|
| AP-UF-01 | 15 flow và thứ tự bước |
| AP-UF-02 | Mapping event → animation (UF-06, UF-12) |
| AP-UF-03 | Hành vi Pause/Disable (UF-10) |
| AP-UF-04 | Hành vi recovery mặc định (UF-15) |
| AP-UF-05 | Giải quyết OI-UF-01…07 |
