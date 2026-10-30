# TRACEABILITY — PART 3 (FR-200 … FR-292)

**Trạng thái:** Draft Phase 0, chưa được Owner duyệt. Nguồn: `FUNCTIONAL_REQUIREMENTS.md`, `ACCEPTANCE_CRITERIA.md`, `AC_ADDENDUM_PART3.md`.
Ghi chú: file FR định nghĩa 61 FR có id >= 200 (không tồn tại FR-202…205, 219…230, 246…260, 285); bảng liệt kê đủ 61 FR, mỗi FR đúng 1 lần.
Status: AC_DEFINED = đã có AC trong ACCEPTANCE_CRITERIA.md; AC_ADDED_PHASE0 = AC bổ sung ở PART3 (AC-401…433); NEEDS_OWNER_CLARIFICATION = cần Owner làm rõ (xem Q-xx trong AC_ADDENDUM_PART3.md); **`AC_PROPOSED_OWNER_DECIDED` (giá trị mới, OWR-3) = có AC PROPOSED (agent viết câu chữ), Owner đã xác nhận quyết định ở OWR-3 nên không còn câu hỏi chặn; AC vẫn là PROPOSED, không phải APPROVED, vì agent không được tự chuyển PROPOSED thành APPROVED; `WONT_MVP` (giá trị mới, OWR-3) = FR bị Owner loại khỏi MVP (WON'T DO IN MVP), giữ hàng truy vết và không viết AC.** `AC_PROPOSED_PENDING_CLARIFICATION` (OWR-2) nay không còn FR nào. Cập nhật OWR-3: FR-200, 283, 284, 292 đã được Owner quyết định; AC-649..AC-657 (PROPOSED).

| FR ID | Priority | Acceptance Criteria ID(s) | Test Scenario | Expected Result | Verification Status |
|---|---|---|---|---|---|
| FR-200 | C | AC-649, AC-650, AC-651 | Approval chờ 24 giờ; alert vs incident; approval hành động quan trọng 15 phút | remind_at = 24 giờ phát đúng 1 operational alert (không phải incident); expires_at = 15 phút là trường riêng **[Nguồn: OWR-3 C, A.12]** | AC_PROPOSED_OWNER_DECIDED |
| FR-201 | S | AC-401 | Mở lịch sử 3 approval đã quyết định | Đủ người, thời điểm, quyết định, feedback; khớp DB | AC_ADDED_PHASE0 |
| FR-206 | M | AC-107 | Agent project A truy cập workspace project B | Thất bại | AC_DEFINED |
| FR-207 | M | AC-101 | Agent chạy lệnh | Lệnh chạy trong Docker container, không chạy trên host | AC_DEFINED |
| FR-208 | M | AC-102 | Lệnh vượt giới hạn CPU/RAM/time | Tiến trình bị dừng, task FAILED/BLOCKED, audit ghi | AC_DEFINED |
| FR-209 | M | AC-103, AC-104 | Lệnh trong deny list; lệnh ngoài allow và deny | Lệnh deny bị chặn; lệnh còn lại chờ approval | AC_DEFINED |
| FR-210 | M | AC-105 | Agent đọc file ngoài workspace | Thất bại, audit ghi | AC_DEFINED |
| FR-211 | M | AC-106 | Agent curl ra ngoài khi allow list rỗng | Thất bại, audit ghi | AC_DEFINED |
| FR-212 | M | AC-032, AC-113 | Quét DB/log/sandbox sau test | 0 credential plaintext, 0 credential trong container | AC_DEFINED |
| FR-213 | M | AC-108 | Owner xem audit log | Mọi sự kiện có timestamp, agent, task; không sửa/xóa từ UI | AC_DEFINED |
| FR-214 | M | AC-109, AC-110 | Emergency Stop với 5 agent Running; Resume | Dừng toàn bộ trong ≤ 5 s, task → BLOCKED; Resume từ checkpoint | AC_DEFINED |
| FR-215 | M | AC-111 | Agent L1 tạo branch/commit | Bị chặn theo permission level, audit ghi | AC_DEFINED |
| FR-216 | S | AC-114 | Docker không chạy, tạo project | Cảnh báo rõ; hành vi fallback chờ quyết định Owner (OI-01) | AC_DEFINED |
| FR-217 | C | AC-402 | Xuất audit log JSON và CSV | Đủ N bản ghi, đúng trường, 0 secret | AC_ADDED_PHASE0 |
| FR-218 | S | AC-403 | Kill switch 1 agent rồi 1 project | Chỉ phạm vi được chọn dừng; agent khác vẫn RUNNING | AC_ADDED_PHASE0 |
| FR-231 | M | AC-063 | Concurrency 5, 8 task READY | Tối đa 5 RUNNING, 3 xếp hàng | AC_DEFINED |
| FR-232 | M | AC-064 | Đặt concurrency = 18 | 18 slot được cấp; rate limit thì backoff | AC_DEFINED |
| FR-233 | M | AC-404 | Seed company mới | Đúng 18 role mặc định theo danh sách | AC_ADDED_PHASE0 |
| FR-234 | M | AC-048 | Stage 5 với 3 task song song | Mỗi task có branch/worktree riêng | AC_DEFINED |
| FR-235 | M | AC-011 | Agent A ghi file thuộc task của B | Bị chặn, audit ghi vi phạm | AC_DEFINED |
| FR-236 | M | AC-405 | Gửi message rồi restart app | Message có cấu trúc, còn nguyên sau restart | AC_ADDED_PHASE0 |
| FR-237 | M | AC-406 | Concurrency 2 với 3 task READY | 2 nhận slot, 1 chờ; token/thời gian được ghi | AC_ADDED_PHASE0 |
| FR-238 | M | AC-407 | Mở monitoring với 3 agent RUNNING | Hiển thị đúng 3 agent, cập nhật ≤ 2 s | AC_ADDED_PHASE0 |
| FR-239 | M | AC-408 | Pause, Resume, Stop, Retry, Reassign | Mỗi thao tác đổi đúng trạng thái và có audit event | AC_ADDED_PHASE0 |
| FR-240 | M | AC-409 | Task phát 100 dòng log/giây trong 5 s | Hiển thị đủ 500 dòng, lọc đúng theo task | AC_ADDED_PHASE0 |
| FR-241 | M | AC-410 | Agent sửa 3 file | Diff đúng 3 file, khớp git diff | AC_ADDED_PHASE0 |
| FR-242 | S | AC-411 | Chọn Open workspace | Mở đúng thư mục workspace; lỗi rõ nếu không có | AC_ADDED_PHASE0 |
| FR-243 | S | AC-049 | Task dev hoàn thành | Task → REVIEWING, merge chỉ sau review pass | AC_DEFINED |
| FR-244 | M | AC-063 | 8 task READY với concurrency 5 | 3 task xếp hàng theo priority | AC_DEFINED |
| FR-245 | S | AC-064 | Provider trả RATE_LIMITED | Backoff, task không FAILED | AC_DEFINED |
| FR-261 | M | AC-412 | Đi qua Welcome/Onboarding | Không qua bước 1 khi provider chưa ping được; tạo agent thật | AC_ADDED_PHASE0 |
| FR-262 | M | AC-413 | Mở Dashboard với 18 agent, 500 task | Needs-you ≤ 2 s; metric có tooltip; load < 1.5 s | AC_ADDED_PHASE0 |
| FR-263 | M | AC-414 | Mở Office, click nhân vật | Mỗi state có hình ảnh riêng; panel 8 trường, 10 action | AC_ADDED_PHASE0 |
| FR-264 | S | AC-415 | Org Chart 18 agent ở 1440 px, kéo thả | Không cuộn ngang; đồng bộ Office ≤ 2 s | AC_ADDED_PHASE0 |
| FR-265 | M | AC-416 | Xóa phòng ban còn agent | Bị chặn; xóa được sau khi chuyển agent | AC_ADDED_PHASE0 |
| FR-266 | M | AC-417 | Lọc, đổi view, Stop all | Card đủ trường; filter giữ; Stop all có xác nhận | AC_ADDED_PHASE0 |
| FR-267 | M | AC-418 | Mở từng tab qua `?tab=` | Đúng 10 tab, deep link hoạt động | AC_ADDED_PHASE0 |
| FR-268 | M | AC-419 | Hoàn tất wizard | 6 bước; agent xuất hiện ≤ 2 s | AC_ADDED_PHASE0 |
| FR-269 | M | AC-420 | Lọc và assign hàng loạt skill | Lọc đúng; tab Skills cập nhật ngay | AC_ADDED_PHASE0 |
| FR-270 | M | AC-421 | Save, Publish, Test run skill | Version mới + diff; xác nhận liệt kê agent; token/latency thật | AC_ADDED_PHASE0 |
| FR-271 | M | AC-422 | Đổi health provider | Dot đổi ≤ 2 s; 0 secret trong DOM | AC_ADDED_PHASE0 |
| FR-272 | M | AC-423 | Tạo project từ prompt | Mở Project Detail với task Discovery RUNNING | AC_ADDED_PHASE0 |
| FR-273 | M | AC-424 | Mở Project Detail khi gate chưa resolved | Phase chưa ✓; deliverable mở được | AC_ADDED_PHASE0 |
| FR-274 | S | AC-425 | Dựng và Run workflow có lỗi validation | Run bị chặn; node cập nhật ≤ 1 s; Undo/Redo 50 bước | AC_ADDED_PHASE0 |
| FR-275 | M | AC-426 | Dùng 4 view Task Board | Chung filter trong URL; cảnh báo cycle | AC_ADDED_PHASE0 |
| FR-276 | M | AC-427 | Filter Needs action, Mark read | Chỉ mục chưa resolved; badge đồng bộ | AC_ADDED_PHASE0 |
| FR-277 | M | AC-428 | Reject không lý do rồi có lý do | Chặn khi thiếu lý do; ghi history; event ≤ 1 s | AC_ADDED_PHASE0 |
| FR-278 | S | AC-429 | Mở file theo deep link khi agent ghi file | Đúng dòng; không mất chỉnh sửa local | AC_ADDED_PHASE0 |
| FR-279 | M | AC-430 | Mở preview, gây lỗi runtime | App thật; lỗi hiển thị; Request changes kèm screenshot | AC_ADDED_PHASE0 |
| FR-280 | M | AC-431 | Click và Dismiss finding | Mở đúng dòng; Dismiss cần lý do; metric thật | AC_ADDED_PHASE0 |
| FR-281 | S | AC-432 | Xem cost với 7 ngày dữ liệu | Tooltip nguồn/giá; forecast chỉ khi ≥ 7 ngày | AC_ADDED_PHASE0 |
| FR-282 | M | AC-433 | Đổi setting, Export diagnostics | Áp dụng đúng thời điểm; 0 secret | AC_ADDED_PHASE0 |
| FR-283 | S | AC-652, AC-653 | Kiểm 1 màn hình theo checklist 10 tiêu chí; thử xóa/hạ mục P0 không bằng chứng | Mỗi tiêu chí Pass/Fail/N/A kèm bằng chứng; P0 không được xóa khi chưa có bằng chứng QA **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |
| FR-284 | C | AC-654, AC-655 | Cài mới; đổi sang tiếng Anh | Mặc định tiếng Việt; hỗ trợ tiếng Anh, lưu qua restart, catalog đủ khóa **[Nguồn: OWR-3 C; B X-1]** | AC_PROPOSED_OWNER_DECIDED |
| FR-286 | M | AC-121, AC-125, AC-126 | Kill app rồi mở lại | Dashboard, approval, hội thoại, message còn nguyên | AC_DEFINED |
| FR-287 | M | AC-121, AC-122 | Kill app, chọn Tiếp tục | Resume từ checkpoint, không chạy lại bước xong | AC_DEFINED |
| FR-288 | M | AC-123 | Provider timeout tạm thời | Retry theo max retry/backoff, hiển thị retry count | AC_DEFINED |
| FR-289 | M | AC-122 | Resume sau restart | Resume từ checkpoint gần nhất, không tạo trùng | AC_DEFINED |
| FR-290 | M | AC-124 | Kill tiến trình CLI từ ngoài | Phát hiện ≤ 60 s; task FAILED hoặc retry | AC_DEFINED |
| FR-291 | S | AC-122 | Resume step đã hoàn tất | Artifact không bị tạo trùng | AC_DEFINED |
| FR-292 | C | AC-656, AC-657 | Rà soát tài liệu; thực hiện dump rồi khôi phục | MVP chỉ có hướng dẫn PostgreSQL dump (đã kiểm khôi phục); không có export workspace **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |

## Tổng kết

| Status | Số FR |
|---|---|
| AC_DEFINED | 24 |
| AC_ADDED_PHASE0 | 33 |
| AC_PROPOSED_OWNER_DECIDED (OWR-3: FR-200, 283, 284, 292) | 4 |
| NEEDS_OWNER_CLARIFICATION | 0 |
| **Tổng** | **61** |
