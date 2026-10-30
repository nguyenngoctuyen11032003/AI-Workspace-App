# ACCEPTANCE CRITERIA ADDENDUM — PART 3 (FR-200 … FR-292)

**Phiên bản:** 0.1 (Phase 0 finalization) · **Tác giả:** BA agent · **Trạng thái:** Draft - CHƯA được Owner duyệt · **Ngày:** 2026-10-30

> Quy ước: cùng định dạng Given/When/Then như `ACCEPTANCE_CRITERIA.md`. AC mới dùng ID AC-401…AC-499 (dải của PART3; PART1/PART2 dùng dải khác). AC chỉ phát biểu lại điều FR đã yêu cầu, không thêm tính năng. Các ngưỡng số đánh dấu [Đề xuất] lấy từ `docs/design/SCREEN_SPECIFICATION.md` hoặc BA đề xuất, cần Owner chốt. Không mục nào ở đây là APPROVED. Đường dẫn code (`apps/web`, `apps/backend`) là giả định theo cách PART2 dùng, chưa có code thật.

## 0. Ghi chú phạm vi và số liệu

- `FUNCTIONAL_REQUIREMENTS.md` định nghĩa **61 FR** có id >= 200: FR-200, 201, 206…218, 231…245, 261…284, 286…292 (không tồn tại FR-202…205, 219…230, 246…260, 285).
- **24 FR** đã có AC trong `ACCEPTANCE_CRITERIA.md` (AC_DEFINED): FR-206…216, 231, 232, 234, 235, 243, 244, 245, 286…291. Không viết lại.
- **33 FR** được bổ sung AC mới (AC-401…AC-433, AC_ADDED_PHASE0): FR-201, 217, 218, 233, 236…242, 261…282.
- **4 FR** NEEDS_OWNER_CLARIFICATION, không có AC: FR-200, 283, 284, 292 (mục 5).
- FR-261…282 là 22 màn hình; AC ở `SCREEN_SPECIFICATION.md` §3.x là checkbox cục bộ không có ID. AC-412…433 gán ID và đo được cho từng màn, dựa trên chính các checkbox đó.

## 1. Approval

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-401 | Đã có 3 approval đã quyết định: 1 Approve, 1 Reject kèm lý do, 1 Request changes kèm feedback | Owner mở lịch sử phê duyệt | Danh sách có đúng 3 dòng; mỗi dòng có đủ: người quyết định, timestamp, quyết định, feedback (lý do khi Reject/Request changes); dữ liệu khớp bản ghi approval trong DB (0 khác biệt); sau khi restart app vẫn đủ 3 dòng | FR-201 | Integration test (API + DB) + E2E Playwright `apps/web` | S |

## 2. Security

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-402 | Audit log có N bản ghi (N >= 50) gồm lệnh, file change, approval, EMERGENCY_STOP | Owner xuất JSON rồi xuất CSV (P2) | Mỗi file có đúng N dòng/bản ghi; mỗi bản ghi chứa timestamp, agent, task, loại sự kiện; file JSON parse được; file CSV có header và số cột nhất quán; không chứa credential (quét regex secret = 0 kết quả) | FR-217 | Integration test + security test (regex secret scan) | C (P2) |
| AC-403 | Project P có agent A1, A2 RUNNING; project Q có agent B1 RUNNING | Owner kích hoạt kill switch cho A1; sau đó kích hoạt kill switch cho project P | Bước 1: chỉ A1 dừng (session hủy, container dừng, task của A1 → BLOCKED), A2 và B1 vẫn RUNNING. Bước 2: A2 dừng, B1 vẫn RUNNING; mỗi bước có 1 audit event ghi phạm vi (agent/project); emergency stop toàn cục (FR-214) không bị kích hoạt | FR-218 | Integration test với FakeProvider + E2E | S |

## 3. Multi-agent

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-404 | Cài mới, company được seed | Truy vấn danh sách role mặc định | Có đúng 18 role: PM, BA, Architect, UI/UX, FE, BE, DB&Infra, QA, Security, Code Reviewer/Product Evaluator, R&D, Integration, Performance, DevSecOps, Documentation, UX Research, Test Automation, Release Manager; không trùng tên; không thiếu, không thừa | FR-233 | Unit test seed + integration test API `apps/backend` | M |
| AC-405 | Agent A gửi message cho agent B và 1 message hệ thống gửi agent | Đọc message từ DB, sau đó kill rồi mở lại app | Mỗi message là bản ghi có cấu trúc (sender, receiver, type, payload, timestamp, correlation id); message sai schema bị từ chối; sau restart số message và nội dung không đổi (0 khác biệt) | FR-236 | Integration test + chaos test (kill process) | M |
| AC-406 | Concurrency = 2; 3 task READY; mỗi task có token budget | Resource manager điều phối | Chỉ 2 task nhận slot, task thứ 3 ở hàng đợi; slot được trả khi 1 task xong và task thứ 3 nhận slot; token đã dùng và thời gian chạy của từng task được ghi nhận, truy vấn được qua API; số slot đang dùng không bao giờ vượt 2 | FR-237 | Integration test với FakeProvider | M |
| AC-407 | 3 agent RUNNING | Owner mở monitoring dashboard | Dashboard liệt kê đúng 3 agent với task hiện tại, tiến độ x/y, log stream; khi 1 agent đổi trạng thái, UI cập nhật trong ≤ 2 s [Đề xuất, SCREEN_SPECIFICATION §3.2]; số agent hiển thị khớp DB | FR-238 | E2E Playwright + WebSocket integration test | M |
| AC-408 | Agent A RUNNING với task T | Owner lần lượt dùng Pause, Resume, Stop, Retry (trên task FAILED), Reassign sang agent B | Pause: A → Paused; Resume: A tiếp tục từ checkpoint; Stop: session hủy, task → BLOCKED/CANCELLED theo state machine; Retry: task chạy lại với retry count + 1; Reassign: task thuộc B, A không còn được gán; mỗi thao tác có 1 audit event; control không hợp lệ với state hiện tại bị ẩn hoặc vô hiệu | FR-239 | E2E + integration test | M |
| AC-409 | Task T của agent A phát 100 dòng log/giây trong 5 s | Owner mở logs của agent A rồi của task T | Hiển thị đủ 500 dòng, không mất dòng (so với log đã lưu); lọc theo task chỉ trả dòng của T; log của agent khác không xuất hiện | FR-240 | E2E + integration test (so khớp số dòng) | M |
| AC-410 | Agent A sửa 3 file trong worktree của task T (1 thêm, 1 sửa, 1 xóa) | Owner mở changes của task T | Diff liệt kê đúng 3 file với trạng thái added/modified/deleted; nội dung diff khớp `git diff` của worktree (0 khác biệt); task không có thay đổi hiển thị danh sách rỗng, không lỗi | FR-241 | Integration test (so với git) + E2E | M |
| AC-411 | Agent A có workspace ở đường dẫn W | Owner chọn Open workspace | Hệ thống mở đúng thư mục W bằng trình quản lý file hệ điều hành; với agent không có workspace hiển thị lỗi rõ ràng và không mở thư mục khác; hành động có audit event. Phần IDE xem Q-05 | FR-242 | Integration test với stub shell-open + manual trên Windows | S |

## 4. UI screens (22 màn hình)

Nguồn chi tiết: `docs/design/SCREEN_SPECIFICATION.md` §3.1…§3.22. Test E2E dùng Page Object theo màn hình (TESTING.md).

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-412 | Cài mới, chưa có provider Connected | Owner đi qua Welcome/Onboarding | Không hoàn tất được bước 1 với provider chưa ping thành công (trừ Skip có cảnh báo); bước 2 tạo agent thật qua `POST /api/agents` và agent xuất hiện trong Office; thoát giữa chừng rồi quay lại thì đúng bước cũ; 0 số liệu hoặc ảnh demo giả | FR-261 | E2E Playwright | M |
| AC-413 | 18 agent, 500 task, WebSocket hoạt động | Owner mở Main Dashboard và hệ thống phát event cần Owner | Mục Needs-you xuất hiện ≤ 2 s sau event; mỗi metric có tooltip nêu công thức/nguồn; mọi progress có mẫu số (x/y); click metric chuyển đến đúng filter; dashboard load < 1.5 s [Đề xuất] | FR-262 | E2E + performance test Playwright | M |
| AC-414 | 5 agent ở 5 state khác nhau, 1 agent blocked | Owner mở Virtual Office và click 1 nhân vật | Mỗi state có biểu diễn hình ảnh riêng và legend; bubble xuất hiện ≤ 1 s sau event blocked; click nhân vật mở panel đủ 8 trường và 10 action; Office lỗi render thì hiện danh sách fallback và app vẫn dùng được; mọi animation transition có event id (đối chiếu log). FPS xem AC-087/AC-095 | FR-263 | E2E + integration test đối chiếu log | M |
| AC-415 | 18 agent, màn 1440 px | Owner mở Org Chart, chế độ Fit, và kéo 1 agent sang phòng ban khác | 18 agent vừa màn không cuộn ngang; dot trạng thái khớp màn Agent Management; kéo thả có hộp xác nhận và đồng bộ với Office Area ≤ 2 s | FR-264 | E2E Playwright (viewport 1440) | S |
| AC-416 | Phòng ban D có 2 agent | Owner tạo, sửa D; xóa D khi còn agent (không chọn phòng đích); sau đó chuyển 2 agent đi và xóa lại | Tạo/sửa lưu tên, mô tả, màu, icon; xóa lần 1 bị chặn với thông báo nêu số agent còn lại (2); xóa lần 2 thành công; thay đổi ánh xạ Area phản ánh ngay trong Office | FR-265 | E2E + integration test | M |
| AC-417 | 18 agent với state khác nhau | Owner mở Agent Management, lọc, đổi view grid/list/org, nhấn Stop all | Card hiển thị đủ name, role, provider, model, current task, state, runtime, last log; đổi view giữ nguyên filter; agent cần chú ý ở đầu danh sách; Stop all hiện hộp xác nhận liệt kê tên từng agent trước khi thực hiện; 18 card cập nhật realtime không layout shift | FR-266 | E2E Playwright | M |
| AC-418 | Agent X tồn tại | Owner mở `/agents/X?tab=<t>` lần lượt cho từng tab | Có đúng 10 tab: Overview, Profile, Skills, Knowledge, Provider, Tasks, Conversations, Logs, Performance, Permissions; deep link `?tab=` mở đúng tab; action bar chỉ hiện action hợp lệ với state; sửa system prompt tạo version mới có diff và rollback; log stream 100 dòng/giây không mất dòng | FR-267 | E2E Playwright | M |
| AC-419 | Provider Connected, có template agent | Owner hoàn tất wizard Agent Builder | Wizard có đúng 6 bước; preview cập nhật theo mỗi field; chưa có dữ liệu thì Est. cost hiển thị "—"; agent tạo xong xuất hiện trong Office và Agents ≤ 2 s; thoát giữa chừng rồi quay lại đúng bước | FR-268 | E2E Playwright | M |
| AC-420 | Thư viện có skill thuộc 6 danh mục; 2 agent | Owner mở Skill Library, lọc theo danh mục, assign hàng loạt skill cho 2 agent | Lọc chỉ trả skill đúng danh mục; thống kê success/usage là dữ liệu thật, hiển thị "—" khi chưa có; sau assign, tab Skills của cả 2 agent hiển thị skill ngay (không cần reload) | FR-269 | E2E Playwright | M |
| AC-421 | Skill S có 1 version, 2 agent đang dùng | Owner sửa và Save, rồi Publish, rồi Test run | Mỗi Save tạo version mới và xem được diff; Publish hiện hộp xác nhận liệt kê 2 agent bị ảnh hưởng; Test run là request thật đến adapter, hiển thị token và latency | FR-270 | E2E + integration test | M |
| AC-422 | Provider Claude Code Connected, Codex Disconnected | Owner mở Provider Hub, health 1 provider đổi trạng thái, sau đó mở màn Cost | Card và topbar dot đổi ≤ 2 s sau khi health đổi; "Switch affected agents" liệt kê agent và kết quả từng agent; secret không bao giờ hiển thị plaintext (quét DOM = 0 secret); số usage khớp màn Cost Management | FR-271 | E2E + security test (DOM/payload scan) | M |
| AC-423 | Có 3 project | Owner mở Project Management, tạo project từ prompt | Card chỉ hiện số liệu có evidence; sau khi tạo, điều hướng tới Project Detail và task Discovery ở trạng thái RUNNING thật | FR-272 | E2E Playwright | M |
| AC-424 | Project có workflow stage 1 xong, gate approval chưa resolved | Owner mở Project Detail | Phase chỉ hiển thị ✓ khi gate approval đã resolved (stage 1 chưa ✓); deliverable là artifact thật, mở được qua link (HTTP 200); Code Workspace, Product Preview, QA & Security Report là tab của Project Detail | FR-273 | E2E Playwright | M |
| AC-425 | Workflow Builder (P2), workflow trống | Owner kéo node từng loại, nối Condition (2 out) và Merge (N in), tạo lỗi validation rồi nhấn Run | Kéo thả và nối cạnh theo quy tắc port; Run bị chặn khi validate lỗi; trạng thái node cập nhật ≤ 1 s sau WS; Undo/Redo 50 bước; deep link `?run=&node=` mở inspector đúng node. Số loại node (9 theo SCREEN_SPECIFICATION so với 8 ở FR-113/AC-306) cần Owner thống nhất, xem Q-06 | FR-274 | E2E Playwright | S (P2) |
| AC-426 | Project có 30 task, 1 cặp task phụ thuộc vòng | Owner mở Task Board ở 4 view kanban/table/timeline/graph | 4 view dùng chung filter và filter nằm trong URL; graph hiển thị critical path và cảnh báo cycle; đánh Done thủ công khi task agent chưa có evidence bị chặn và hiện lý do; 500 task ở table scroll không lỗi (virtualized) | FR-275 | E2E Playwright | M |
| AC-427 | 10 event gồm 3 mục cần hành động | Owner mở Activity Center, bật filter Needs action, nhấn Mark read | Filter chỉ hiện đúng 3 mục chưa resolved; mỗi dòng có deep link đúng đích; Mark read cập nhật badge sidebar đồng bộ | FR-276 | E2E Playwright | M |
| AC-428 | 2 approval WAITING | Owner mở Approval Center, thử Reject không lý do, rồi Reject có lý do, rồi nhấn phím `A` | Reject/Request changes thiếu lý do không gửi được; quyết định có lý do ghi history (ai, khi nào, lý do) và gửi event về workflow ≤ 1 s; phím `A/X/M` chỉ mở form tương ứng, không áp dụng ngay; checks hiển thị pending/running/done thật, không có ✓ khi chưa chạy | FR-277 | E2E Playwright + integration test | M |
| AC-429 | Project có workspace; agent đang ghi file F đang mở trong editor với thay đổi chưa lưu | Owner mở Code Workspace bằng deep link `?file=F&line=42` | Editor mở đúng file và dòng 42; khi agent ghi F, editor cập nhật, hiện cảnh báo, giữ bản local chưa lưu và có diff (0 chỉnh sửa bị mất); commit của Director và agent phân biệt được; terminal chạy trong sandbox với giới hạn quyền như agent | FR-278 | E2E Playwright | S |
| AC-430 | Project đã build, app preview chạy trong sandbox | Owner mở Product Preview, gây lỗi runtime, nhấn Request changes | Preview là app thật (đổi code → UI đổi sau reload; không phải ảnh tĩnh); console lỗi runtime hiển thị trong app; Request changes mang theo URL và screenshot | FR-279 | E2E Playwright | M |
| AC-431 | Có 3 QA run, 1 finding Critical | Owner mở QA & Security Report, click finding, Dismiss | Finding mở file đúng dòng trong Code Workspace; Dismiss bắt buộc nhập lý do và lưu history; metric là số từ run thật, "—" khi chưa chạy; flaky chỉ gán khi có ≥ 3 run; finding Critical mới tạo badge sidebar và Needs-you | FR-280 | E2E Playwright | M |
| AC-432 | Có 7 ngày dữ liệu usage; ngân sách phòng ban bị vượt | Owner mở Cost Management, lọc theo from/to và groupBy | Mỗi con số có tooltip nêu nguồn (provider usage) và bảng giá áp dụng (ngày); forecast chỉ hiện khi có ≥ 7 ngày dữ liệu và ghi phương pháp; hành động khi vượt ngân sách (pause tasks) thực thi đúng và có log | FR-281 | E2E + integration test | S |
| AC-433 | App chạy | Owner đổi 1 setting hiển thị (theme) và 1 setting thực thi (concurrency), rồi Export diagnostics | Theme áp dụng ngay; setting concurrency hiển thị "áp dụng từ task tiếp theo" và chỉ có hiệu lực từ task kế; file diagnostics chứa 0 secret (quét regex) | FR-282 | E2E + security test (regex scan) | M |

## 5. Cần Owner làm rõ

**Cập nhật OWR-3 (2026-10-30):** Q-01 (FR-200, N = 24 giờ, operational alert; remind_at/expires_at), Q-02 (FR-283, checklist 10 tiêu chí), Q-03 (FR-284, tiếng Việt mặc định, hỗ trợ tiếng Anh), Q-04 (FR-292, chỉ hướng dẫn PostgreSQL dump) đã được Owner quyết định ở OWR-3 mục C; AC-649..AC-657 trong `AC_ADDENDUM_PART4_OWR2.md` (PROPOSED). Q-06 (số loại node): 9 UI node type theo OWR-3 A.14. Bảng gốc giữ bên dưới để truy vết.

| ID | FR | Câu hỏi cụ thể | Lý do chưa viết AC |
|---|---|---|---|
| Q-01 | FR-200 | "N giờ" là bao nhiêu (giá trị mặc định, có cấu hình theo approval/policy không)? Nhắc nhở gửi tới ai, qua kênh nào (in-app/OS notification), lặp lại mấy lần, và "escalation" nghĩa là gì (chuyển người duyệt khác hay tự reject/hủy)? | Không có ngưỡng, người nhận và hành động leo thang để đo kết quả |
| Q-02 | FR-283 | "Premium" đo bằng tiêu chí nào: bộ token trong `DESIGN_TOKENS.md` có là chuẩn nghiệm thu không? Mức tương phản tối thiểu (WCAG AA?), ngân sách animation (ms), ngưỡng visual regression (TESTING.md: 0.2%) có được chấp nhận làm AC không? Dark và light phải đủ cho mọi màn hình? | "Premium", "tinh tế" mang tính chủ quan, không kiểm thử được nếu không có tiêu chí |
| Q-03 | FR-284 | OI-07 chưa chốt: hỗ trợ Việt, Anh hay song ngữ; ngôn ngữ mặc định; chuỗi nào phải dịch (UI, lỗi, log agent); định dạng ngày/số theo locale? | FR ở trạng thái [Chưa chốt]; không có phạm vi để đo (bao nhiêu % chuỗi đã dịch) |
| Q-04 | FR-292 | "Sao lưu/xuất dữ liệu workspace" gồm những gì (DB, repo, artifact, config; loại trừ credential?), định dạng file (zip/JSON/SQLite), đích lưu, có cần restore/import, có mã hóa, có lịch tự động? | Phạm vi và định dạng chưa xác định |
| Q-05 | FR-242 | (Không chặn, AC-411 đã viết) "Mở thư mục/IDE": IDE mặc định là gì (VS Code, cấu hình được)? | AC-411 chỉ kiểm mở thư mục bằng trình quản lý file; phần IDE chờ xác nhận |
| Q-06 | FR-274 | (Không chặn, AC-425 đã viết) SCREEN_SPECIFICATION nêu 9 loại node, FR-113 nêu 8 loại (AC-306): số loại node chuẩn là bao nhiêu? PRD §8 nói MVP chỉ ~15 màn hình cốt lõi: AC-412…433 áp dụng cho bản functional của màn nào ở MVP? | Mâu thuẫn số loại node; phạm vi MVP của các màn UI chưa chốt |

## 6. Số liệu

- AC mới: 33 (AC-401…AC-433). Dải AC-434…AC-499 để trống, dự trữ.
- Câu hỏi Owner: 6 (4 chặn AC: Q-01…Q-04; 2 không chặn: Q-05, Q-06).
