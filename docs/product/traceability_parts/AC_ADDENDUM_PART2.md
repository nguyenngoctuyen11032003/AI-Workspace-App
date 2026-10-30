# ACCEPTANCE CRITERIA ADDENDUM — PART 2 (FR-101 … FR-199)

**Phiên bản:** 0.1 (Phase 0 finalization) · **Tác giả:** BA agent · **Trạng thái:** Draft - CHƯA được Owner duyệt · **Ngày:** 2026-10-30

> Quy ước: cùng định dạng Given/When/Then như `ACCEPTANCE_CRITERIA.md`. AC mới dùng ID AC-301…AC-399 (dải của PART2; PART1 dùng dải khác). AC-097…AC-100 là mã đã bị tham chiếu nhưng chưa định nghĩa, được định nghĩa với đúng ID cũ. AC chỉ phát biểu lại điều FR đã yêu cầu; không thêm tính năng. Giá trị số [Đề xuất] cần Owner chốt. Không mục nào ở đây là APPROVED.

## 0. Ghi chú phạm vi và số liệu

- `FUNCTIONAL_REQUIREMENTS.md` hiện **chỉ định nghĩa 62 FR** trong dải FR-101…FR-199 (FR-101…135, 151…165, 181…188, 196…199). Các ID FR-136…150, 166…180, 189…195 không tồn tại trong file FR (dải dự trữ). Yêu cầu "99 FR" không khớp thực tế; PART2 phủ đủ 62 FR có thật. Đây là điểm cần Owner/QA xác nhận (xem Q-00).
- Dải AC thiếu nhưng được trích dẫn rõ ràng ở `IMPLEMENTATION_ROADMAP.md` (dòng 45, 350: "AC-086…100"): AC-097…100. Đã định nghĩa ở mục 1.
- Dải AC thiếu chỉ xuất hiện trong khung "AC-001…AC-130" (TESTING.md:11, ROADMAP:5) mà không có ngữ cảnh trích dẫn: AC-065…070, AC-083…085, AC-115…120. Không suy diễn ý định; không định nghĩa ở PART2 (AC-083…085 thuộc nhóm Skill, AC-115…120 thuộc nhóm Security ngoài slice FR-101…199). AC-065…070 cũng không có FR slice 101…199 nào thiếu AC cần mã đó; các FR thiếu AC đã được gán AC-301+.

## 1. AC bị tham chiếu nhưng chưa định nghĩa (được định nghĩa với ID gốc)

Nguồn trích dẫn: `IMPLEMENTATION_ROADMAP.md` dòng 45 (Phase 6, "FR-151…195 / AC-086…100"), dòng 338-350 (P6-02, P6-04: FR-158-162 / FR-163-165). **Giả định về ý định gốc:** AC-097…100 là các AC của Phase 6 còn thiếu cho FR-162, FR-163, FR-164, FR-165 (các FR của nhóm E chưa có AC nào); gán 1-1 theo thứ tự FR.

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-097 | Bảng ánh xạ event → animation/vị trí đang có mục `TASK_STARTED → Walk tới desk → Typing`; Virtual Office mở | Owner sửa mục `TASK_STARTED` thành `Walk tới desk → Reading` trong cấu hình rồi lưu; sau đó phát event TASK_STARTED thật cho agent X | Nhân vật X thực hiện chuỗi Walk rồi Reading (không còn Typing); khởi động lại app thì cấu hình vẫn giữ; event type không có mục ánh xạ không gây animation nào | FR-162 | Integration test (Vitest + Phaser harness `apps/web/src/office`) + E2E-06 | S |
| AC-098 | Có agent thuộc phòng ban mặc định (vd. Frontend) và 1 phòng ban tùy chỉnh "Data" (P2) | (a) Tạo agent phòng ban Frontend; (b) gán phòng ban "Data" vào phòng DB hiện có | (a) Desk của agent nằm trong phòng FE; (b) agent của "Data" có desk trong phòng DB; bảng ánh xạ phòng ban↔phòng hiển thị cả hai; MVP: ánh xạ 14 phòng mặc định luôn đủ (14/14). Phần "phòng mới" là P2, không kiểm trong MVP | FR-163 | Integration test + E2E-06 | M |
| AC-099 | Agent A có trạng thái Idle/Available (event AGENT_STATE_CHANGED → Idle đã phát) | (a) Không có event Idle nào cho agent B; (b) event AGENT_STATE_CHANGED → Idle phát cho agent A | (a) B không xuất hiện trong phòng Relax/Coffee; (b) A di chuyển tới Relax hoặc Coffee sau event; không nhân vật nào vào Relax/Coffee mà event store thiếu event tương ứng (đối chiếu log event với di chuyển) | FR-164 | Integration test đối chiếu event log với position log; E2E-06 | S |
| AC-100 | Phòng QA có 2 agent và 3 task; Virtual Office mở | Owner click vào phòng QA | Panel liệt kê đúng 2 agent và 3 task của phòng (khớp truy vấn DB); click vào phòng rỗng hiển thị danh sách rỗng (0 agent, 0 task), không lỗi | FR-165 | E2E (Playwright) so khớp với API | S |

## 2. AC mới (AC-301 …)

### 2.1 Workflow & Orchestration

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-301 | Workflow 3 node tuần tự, node 1 hoàn tất | Kill tiến trình backend ngay sau node 1 rồi mở lại | Bản ghi trạng thái workflow trong DB có node 1 = COMPLETED và output; không có trạng thái nào chỉ nằm trong bộ nhớ (kiểm bằng so sánh DB trước và sau restart: 0 khác biệt) | FR-101 | Integration test + chaos test (task kill) ở `apps/backend` | M |
| AC-302 | Workflow có node Condition với 2 nhánh A (kết quả = true) và B (false) | Chạy workflow 2 lần với input làm Condition = true rồi = false | Lần 1 chỉ node thuộc nhánh A thực thi, B ở trạng thái skipped; lần 2 ngược lại; sự kiện STAGE_STARTED chỉ phát cho nhánh được chọn | FR-104 | Integration test với FakeProvider | M |
| AC-303 | Workflow có node Parallel 3 nhánh rồi node Merge; nhánh 3 chậm hơn nhánh 1, 2 | Chạy workflow | Node Merge không bắt đầu cho tới khi cả 3 nhánh COMPLETED; sau đó Merge bắt đầu đúng 1 lần; nếu 1 nhánh FAILED hết retry thì Merge không chạy và workflow hiển thị nhánh lỗi | FR-105 | Integration test (FakeProvider có độ trễ) | M |
| AC-304 | Owner tạo task qua API/UI | Đọc task đã lưu | Task chứa đủ các trường: id, title, description, project, workflow/stage, assignee, status, priority, dependencies, inputs, outputs/artifacts, file ownership, token budget, time budget, retry count, parent task, version, created/updated (18 trường; danh sách là [Đề xuất]); thiếu title hoặc project thì tạo bị từ chối với mã lỗi validation | FR-107 | Unit test schema + API contract test | M |
| AC-305 | Màn Workflow Builder mở, workflow trống (P2) | Owner kéo 2 node vào canvas và nối cạnh | 2 node hiện ở vị trí thả, 1 cạnh có hướng nối node nguồn → đích; lưu rồi mở lại giữ nguyên vị trí và cạnh; nối tạo vòng lặp trực tiếp tự thân bị chặn là phạm vi chưa nêu, không kiểm | FR-112 | E2E (Playwright) | S (P2) |
| AC-306 | Palette Workflow Builder (P2) | Owner mở palette và thả từng loại | Palette có đúng 8 loại: Agent, Approval, Condition, Parallel, Merge, Tool, Test, Deployment; mỗi loại thả được 1 node và lưu được; loại thứ 9 không tồn tại | FR-113 | E2E (Playwright) + unit test registry | S (P2) |
| AC-307 | Workflow template với 3 node Agent (P2, Could) | Owner chạy Dry-run | 0 lời gọi tới provider adapter (đếm lời gọi = 0); kết quả hiển thị trình tự node dự kiến của đường đi và nhánh Condition được mô phỏng; không có task thật hoặc file thay đổi trong workspace | FR-115 | Integration test với adapter spy | C (P2) |
| AC-308 | Hệ thống chạy kịch bản bao phủ vòng đời | Truy vấn registry và event store | Registry có đúng 26 event type của danh mục FR-117 (AGENT_CREATED … PROVIDER_DISCONNECTED); mỗi type có contract test phát ra ít nhất 1 event hợp lệ khớp schema FR-116; event type ngoài registry bị từ chối khi ghi | FR-117 | Contract test + integration test tại `apps/backend` | M |
| AC-309 | Workflow RUNNING có 2 task RUNNING và 1 READY | Owner nhấn Cancel ở cấp workflow | Workflow → CANCELLED; 2 task RUNNING bị hủy phiên provider trong ≤ 5 s [Đề xuất] và → CANCELLED; task READY không còn được gán; có event; Resume sau Cancel không khả dụng | FR-133 | E2E + integration test | M |
| AC-310 | Workflow SDLC đã chạy qua stage 1-3 với 1 approval | Owner mở Timeline workflow | Mỗi giai đoạn hiển thị: tên, thời điểm bắt đầu và kết thúc, agent thực hiện, kết quả (completed/failed/rejected), thông tin approval (người, quyết định); số dòng timeline = số giai đoạn đã chạy; khớp event store | FR-134 | E2E + so khớp với event store | S |

### 2.2 Virtual Office (phần P2)

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-311 | Virtual Office ở chế độ edit layout (P2) | Owner kéo phòng sang vị trí mới và đổi kích thước | Phòng ở vị trí và kích thước mới; reload vẫn giữ; desk của agent nằm trong phòng vẫn nằm trong biên phòng mới | FR-154 | E2E (Playwright) + unit test layout | C (P2) |
| AC-312 | Phòng có ≥ 1 furniture (P2) | Owner đặt thêm 1 furniture rồi kéo sang ô khác | Furniture hiện ở ô đích, lưu khi reload; không đặt chồng lên vị trí ô bị chặn (tường) | FR-155 | E2E (Playwright) | C (P2) |

### 2.3 Dashboards

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-313 | 2 workflow RUNNING, 1 đang chờ approval 10 phút | Owner mở Workflow dashboard | Liệt kê đúng 2 workflow đang chạy; mỗi dòng có giai đoạn hiện tại khớp DB; workflow chờ approval hiển thị thời gian chờ ≈ 10 phút (sai số ≤ 1 phút) | FR-184 | E2E + so khớp API/DB | S |
| AC-314 | Có usage record cho 2 project, 3 agent, 2 phòng ban, 2 provider qua 3 ngày | Owner mở Cost dashboard và đổi lần lượt nhóm theo project / agent / phòng ban / provider và theo thời gian | Với mỗi cách nhóm, tổng các nhóm = tổng toàn bộ usage record (sai số 0); lọc 1 ngày chỉ gồm record ngày đó; record ước lượng gắn nhãn "ước lượng" (FR-096). Đơn vị tiền tệ/token chờ Owner chốt (OI-FR-07) | FR-185 | Integration test aggregate + E2E | M |
| AC-315 | Docker, 2 provider, hàng đợi có 3 task, concurrency 5 với 2 slot đang dùng | Owner mở System dashboard | Hiển thị trạng thái từng provider (Connected/Not installed/...), trạng thái sandbox (Docker chạy/không), số task trong hàng đợi = 3, concurrency đang dùng = 2/5; ngắt Docker thì trạng thái sandbox đổi trong ≤ 10 s [Đề xuất] | FR-187 | E2E + integration test | S |
| AC-316 | Bản build release, workspace mới chưa có dữ liệu | Mở lần lượt tất cả dashboard (FR-181…187 tồn tại) | Mọi con số/biểu đồ hiển thị 0 hoặc empty state; không có số liệu mẫu; quét bundle release và seed không thấy fixture/placeholder dashboard (0 kết quả quét); tạo dữ liệu thật thì số liệu đổi đúng theo | FR-188 | Build scan + E2E trên bản release | M |

### 2.4 Approval policy

| ID | Given | When | Then | FR | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-317 | Policy cho stage 3 đang tắt approval | Owner bật approval cho stage 3 ở Settings rồi chạy workflow | Stage 3 hoàn thành → task WAITING_APPROVAL và APPROVAL_REQUESTED xuất hiện; tắt lại thì stage 3 hoàn thành không chờ; thay đổi policy có trong audit log | FR-198 | Integration test + E2E | M |
| AC-318 | Policy bật cho 4 loại hành động: deploy, xóa file, lệnh ngoài allow list, vượt ngân sách | Agent thực hiện lần lượt 4 hành động | Mỗi hành động tạo đúng 1 yêu cầu approval với loại tương ứng và không thực thi cho tới khi approve; reject thì hành động không chạy; tắt policy của 1 loại thì loại đó không tạo approval (trừ production deploy, xem AC-320) | FR-198 | Integration test với FakeProvider + sandbox thật | M |
| AC-319 | Cài mới, chưa chỉnh policy | Owner mở Settings > Approval policy và chạy SDLC | Stage 2, 4, 8 ở trạng thái bắt buộc approval và chạy dừng ở cả 3 gate; không dừng ở stage khác theo mặc định | FR-199 | E2E + kiểm seed config | M |
| AC-320 | Policy mặc định | Owner (UI) và API cố tắt approval cho production deploy; Agent kích hoạt production deploy | UI không cho tắt (điều khiển bị vô hiệu); API trả lỗi từ chối và không đổi policy; production deploy luôn tạo yêu cầu approval; có audit log về lần cố tắt | FR-199 | Security test + integration test | M |

## 3. Cần Owner làm rõ

Các FR sau quá mơ hồ để viết AC kiểm thử được; không bịa tiêu chí.

| # | FR | THIẾU THÔNG TIN - cần Owner làm rõ |
|---|---|---|
| Q-00 | Phạm vi | FR-136…150, 166…180, 189…195 không tồn tại; xác nhận là dải dự trữ (không cần AC) hay FR bị thiếu cần bổ sung |
| Q-01 | FR-122 | Giai đoạn System Design: "thiết kế hệ thống" gồm những tài liệu/artifact bắt buộc nào (tên, định dạng) và có approval gate hay không (FR-199 chỉ liệt kê stage 2, 4, 8)? |
| Q-02 | FR-153 | Số tầng tối đa và cách tầng gắn với phòng/agent; tiêu chí chuyển tầng (UI, hiệu năng) là gì? (P2/Could) |
| Q-03 | FR-181 | "Sự cố" gồm những loại nào (task FAILED, provider disconnected, sandbox down, EMERGENCY_STOP...) và đếm trong khoảng thời gian nào? |
| Q-04 | FR-182 | "Tiến độ theo giai đoạn" tính thế nào (số giai đoạn xong / 8, hay theo % task)? "Thời gian" là thời gian chạy thực hay lịch? |
| Q-05 | FR-183 | "Hiệu suất" agent đo bằng chỉ số nào (task hoàn thành/giờ, tỷ lệ reject, ...) và công thức? |
| Q-06 | FR-186 | Định nghĩa "test pass rate", "security findings" (đếm theo mức độ nào?), "reject rate" (mẫu số là gì) và cửa sổ thời gian |

## 4. Thống kê

- AC mới AC-301…AC-320: 20 AC. AC được định nghĩa với mã cũ: AC-097…AC-100 (4 AC). Tổng thêm: 24.
- Câu hỏi cần Owner làm rõ: 7 (Q-00…Q-06); 6 FR bị gắn NEEDS_OWNER_CLARIFICATION (FR-122, 153, 181, 182, 183, 186).
- Tổng hợp mã AC đã định nghĩa theo mã cũ: AC-097 → FR-162; AC-098 → FR-163; AC-099 → FR-164; AC-100 → FR-165.
