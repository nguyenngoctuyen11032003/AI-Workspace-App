# ACCEPTANCE CRITERIA ADDENDUM — PART 4 (cập nhật OWR-2) — AI WORKSPACE

**Phiên bản:** 0.1 · **Vai trò:** BA · **Trạng thái:** DRAFT. Toàn bộ AC trong file này là **PROPOSED**, chưa có AC nào được Owner duyệt (không có AC nào APPROVED) · **Ngày:** 2026-10-30
**Nguồn:** `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md` (§1 Q-18 OWN-AC-1..3, §4 FR-122/181/182/183). Phase 0 chưa hoàn tất; Phase 1 chưa được Owner cho phép.

> Quy ước: Given/When/Then như `ACCEPTANCE_CRITERIA.md`. Dải mã dùng: AC-601..AC-614 (bốn FR dashboard/workflow), AC-615..AC-620 dự trữ, AC-621..AC-623 (D-021). Nội dung OWR-2 §4 là **đề xuất của Owner, không phải trích nguyên văn FR**; AC chỉ diễn đạt lại đề xuất đó, không thêm phạm vi. Câu hỏi Owner chưa trả lời giữ nguyên là PENDING_OWNER_CLARIFICATION (OQ-xxx), AC không quyết định thay Owner.

## 1. FR-122, FR-181, FR-182, FR-183 (AC_PROPOSED_PENDING_CLARIFICATION)

### 1.1 FR-122
Văn bản gốc trong repo: "Giai đoạn 3 System Design: Architect agent tạo thiết kế hệ thống." Đề xuất Owner gọi là "Workflow Deliverables" và áp dụng cho Workflow Run. Phạm vi áp dụng (chỉ Stage 3 hay mọi run) là OQ-122-3.

| ID | FR | Given | When | Then | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-601 | FR-122 | Một Workflow Run đã chạy (có node outputs, run state và log) | Owner mở deliverable của run | Deliverable package gồm node outputs, run state, logs và liên kết tới các generated artifact của run đó; truy xuất được theo workflow run id | Integration test + E2E; đối chiếu DB | M |
| AC-602 | FR-122 | Một run tạo ra artifact | Owner mở một artifact trong package | Artifact có metadata (tối thiểu: nguồn tạo ra, tức node/agent/task và thời điểm tạo, gắn với run) và dữ liệu khớp execution evidence; không có artifact không có nguồn | Integration test | M |
| AC-603 | FR-122 | Một run có node thất bại | Owner mở deliverable của run | Run được đánh dấu thất bại và chỉ ra node gây lỗi | Integration test với node lỗi cố ý | M |
| AC-604 | FR-122 | Một run còn ít nhất một required node chưa hoàn thành | Run kết thúc luồng hoặc có yêu cầu đánh dấu hoàn thành | Run không được đánh dấu hoàn thành; chỉ hoàn thành khi mọi required node hoàn thành | Integration test | M |

PENDING_OWNER_CLARIFICATION (không quyết định ở đây):
- OQ-122-1: artifact nào là bắt buộc (AC-601, 602 hiện chưa liệt kê tên artifact bắt buộc).
- OQ-122-2: có yêu cầu export/download không, hay chỉ lưu và truy xuất trong workspace (AC-601 chỉ kiểm lưu và truy xuất).
- OQ-122-3: đề xuất nói mọi Workflow Run, FR gốc chỉ nói Stage 3; phạm vi áp dụng?

### 1.2 FR-181
Văn bản gốc: "Overview dashboard: số agent theo trạng thái, project đang chạy, task theo status, approval chờ, chi phí hôm nay/tháng, sự cố." Đề xuất Owner gọi là "Incident Dashboard".

| ID | FR | Given | When | Then | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-605 | FR-181 | Có sự kiện vận hành bất thường thuộc các loại: execution error, timeout, provider rate limit, sandbox failure, lost connection/heartbeat, approval timeout | Sự kiện xảy ra và có bằng chứng trong event store | Dashboard hiển thị incident có đủ ID, type, severity, timestamp, target (task/agent/workflow/infrastructure bị ảnh hưởng) và status | Integration test mỗi loại + E2E | M |
| AC-606 | FR-181 | Không có bằng chứng sự cố trong event store (workspace mới hoặc hệ thống bình thường) | Owner mở dashboard | Không có incident nào được tạo hoặc hiển thị; không suy diễn incident khi thiếu bằng chứng; hiển thị 0 hoặc empty state | Integration test; liên kết AC-316 | M |
| AC-607 | FR-181 | Cùng một lỗi lặp lại nhiều lần | Hệ thống ghi nhận các lần lặp | Các lần lặp được gom theo **quy tắc grouping**. **ĐỀ XUẤT của BA, chưa phải quyết định:** cùng `type` và cùng `target`, trong cùng cửa sổ N phút (đề xuất N = 15) khi incident trước còn `open`, thì gom thành một incident có `occurrence_count`, `first_seen`, `last_seen`; khi incident đã `resolved` hoặc quá cửa sổ thì tạo incident mới | Unit test của hàm grouping + integration | M |

PENDING_OWNER_CLARIFICATION:
- OQ-181-1: approval chờ lâu là incident hay chỉ operational alert? (`approval timeout` có trong danh sách đề xuất của Owner nhưng chính Owner hỏi lại điểm này; AC-605 phụ thuộc câu trả lời đối với loại này.)
- OQ-181-2: quy tắc grouping ở AC-607 (loại khóa gom, cửa sổ N, khi nào mở incident mới) cần Owner duyệt hoặc thay bằng quy tắc khác.

### 1.3 FR-182
Văn bản gốc: "Project dashboard: tiến độ theo giai đoạn, task, bug, chi phí, thời gian." Đề xuất Owner gọi là "Progress Calculation" và tính theo task.

| ID | FR | Given | When | Then | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-608 | FR-182 | Một project có baseline gồm các task với trọng số | Hệ thống tính tiến độ | Progress = 100 × (tổng trọng số task đã hoàn thành / tổng trọng số task trong baseline); kết quả khớp phép tính độc lập từ DB | Unit test công thức + integration | M |
| AC-609 | FR-182 | Baseline có task `completed`, `running`, `blocked` | Hệ thống tính tiến độ | Chỉ task `completed` được tính là hoàn thành; `running` và `blocked` không được tính | Unit test | M |
| AC-610 | FR-182 | Baseline có task `cancelled` | Hệ thống tính tiến độ | Task `cancelled` được xử lý theo **quy tắc riêng do Owner duyệt** (OQ-182-1); AC này chưa thể kiểm thử cho đến khi quy tắc được chốt | Unit test (viết sau khi có quy tắc) | M |
| AC-611 | FR-182 | MVP chưa có hệ thống estimate đáng tin cậy | Hệ thống tính tiến độ | Mọi task trong baseline có trọng số bằng nhau (nếu Owner chấp thuận, OQ-182-2) | Unit test | M |

PENDING_OWNER_CLARIFICATION:
- OQ-182-1: quy tắc xử lý task `cancelled` (ví dụ loại khỏi mẫu số, hoặc tính như chưa xong; chưa chọn).
- OQ-182-2: có chấp thuận trọng số bằng nhau không.
- OQ-182-3: định nghĩa "baseline" (thời điểm chốt, task thêm hoặc bớt sau baseline).
- OQ-182-4: FR gốc nói "tiến độ theo giai đoạn", đề xuất tính theo task; quan hệ giữa hai cách tính?

### 1.4 FR-183
Văn bản gốc: "Agent dashboard: hiệu suất, token, cost, lỗi, thời gian làm việc." Đề xuất Owner gọi là "Performance Metrics".

| ID | FR | Given | When | Then | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-612 | FR-183 | Có dữ liệu thực thi thật | Owner mở dashboard hiệu suất | Hiển thị lead time, execution time, success rate, retry rate, throughput; các chỉ số này tách biệt với Progress (không dùng Progress làm Performance) | E2E + đối chiếu truy vấn DB độc lập | M |
| AC-613 | FR-183 | Có dữ liệu theo nhiều agent, task, workflow và thời gian | Owner chọn breakdown | Chỉ số được tách theo Agent, Task, Workflow và theo thời gian; tổng các nhóm khớp tổng chung | Integration test | M |
| AC-614 | FR-183 | Provider có hoặc không cung cấp usage | Owner mở dashboard | Token và cost chỉ hiển thị khi provider cung cấp dữ liệu; nếu không có thì hiển thị trạng thái không có dữ liệu (không tự bịa). Giá trị ước lượng, nếu có theo FR-096, phải được đánh dấu "ước lượng" | Integration test | M |

PENDING_OWNER_CLARIFICATION:
- OQ-183-1: dashboard nhiều metric hay một KPI tổng hợp? AC-612..614 giả định nhiều metric theo đề xuất; chưa có AC cho KPI tổng hợp.
- OQ-183-2: nếu chọn KPI tổng hợp thì cần công thức và trọng số được Owner phê duyệt (BA không đề xuất công thức).

## 2. D-021 (APPROVED_CONDITIONAL): OWN-AC-1..3

D-021 chỉ được xem là chấp thuận có điều kiện cho đến khi ba tiêu chí được kiểm thử thành công (OWR-2 Q-18). FR-293..FR-298 là **đề xuất chờ Owner duyệt** (xem `FUNCTIONAL_REQUIREMENTS.md` mục 3.13). Phương pháp kiểm chứng là PoC SK-01..SK-05 (OWR-2 §5.5, scratch, không sửa Skill production, không gọi API ngoài, không dùng secret thật). Trạng thái PoC: **NOT RUN** (BA không có bằng chứng chạy).

| ID | Gốc | FR đề xuất | Given | When | Then | PoC kiểm chứng | Pri |
|---|---|---|---|---|---|---|---|
| AC-621 | OWN-AC-1 Dynamic Skill Loading | FR-293, FR-294, FR-295 | Agent có thư mục `.ai/skills/<agent_name>/` | Owner/Admin thêm hoặc sửa một file Markdown trong thư mục đó, rồi giao task mới | Task mới tự động nhận nội dung Skill mới (manifest và hash khớp file); không cần restart ứng dụng và không cần recompile | SK-01 (sửa Skill, task mới nhận nội dung mới); SK-02 (không restart, không recompile) | M |
| AC-622 | OWN-AC-2 Unauthorized Overlay Modification | FR-296 (và FR-293) | Agent đang chạy; Agent dùng Bash hoặc công cụ khác để đổi file overlay ngoài phạm vi quyền được phép | Hệ thống phát hiện thay đổi | Hệ thống ghi audit log; đánh dấu thay đổi là đáng ngờ; tạm dừng Agent cho đến khi Manager/Owner xác nhận; Agent không tự tiếp tục trước khi có xác nhận. **Giới hạn trung thực bắt buộc (Owner):** vì Owner và Agent có thể dùng cùng Windows OS user, hệ thống KHÔNG được tuyên bố xác định danh tính người sửa file; chỉ được xác định thay đổi, thời điểm, phiên thực thi và bằng chứng audit liên quan (trường "người sửa" để trống hoặc ghi "không xác định") | SK-04 (Agent sửa overlay bằng host Bash: audit log và tạm dừng); SK-05 (Owner xác nhận thì Agent mới được tiếp tục) | M |
| AC-623 | OWN-AC-3 Resume Session Context Refresh | FR-296 (và FR-294) | Một Claude session bị gián đoạn và được resume; Skill hoặc System Prompt đã đổi | Hệ thống kiểm tra phiên bản/hash của Skill và System Prompt khi resume | Nếu nội dung đã đổi thì không tiếp tục dùng context cũ mà không kiểm tra; có cơ chế tạo session mới hoặc refresh context đã được PoC chứng minh hoạt động; ghi audit log việc phát hiện thay đổi và cách xử lý context | SK-03 (resume sau khi Skill/System Prompt đổi: phát hiện hash đổi và refresh context) | M |

Điều kiện chung của PoC (OWR-2 §5.5): giới hạn trong scratch, hạn chế tiêu thụ quota, báo cáo rõ giới hạn phân biệt Owner và Agent khi dùng chung Windows OS user.

PENDING_OWNER_CLARIFICATION (D-021):
- OQ-D021-1: OWN-AC-2 của Owner kích hoạt khi **Agent** đổi file overlay ngoài quyền. Thiết kế `DYNAMIC_SKILL_INJECTION.md` §12.1 kích hoạt rộng hơn: bất kỳ thay đổi hash khi có task RUNNING, kể cả do Owner tự sửa (vì không phân biệt được người sửa), và chuyển task sang `WAITING_APPROVAL`. Cần Owner xác nhận cách này là chấp nhận được (thiết kế đã ghi là "đúng ý Owner" nhưng OWR-2 không nói rõ).
- OQ-D021-2: Owner duyệt FR-293..FR-298 và priority chưa.

## 3. Đối chiếu với AC-501..AC-515 trong `DYNAMIC_SKILL_INJECTION.md` (không đánh số lại)

AC-501..AC-515 là AC tạm trong tài liệu kiến trúc (PROPOSED, chưa vào AC chính thức). Giữ nguyên mã. AC-621..623 là AC cấp Owner; AC-501..515 là AC chi tiết hỗ trợ.

| AC cấp Owner | AC chi tiết trong design (giữ nguyên mã) | Ghi chú đối chiếu |
|---|---|---|
| **AC-621** (OWN-AC-1) | AC-501 (thêm file, task mới nạp), AC-502 (sửa file, task mới dùng hash mới), AC-503 (xóa file), AC-504 (sửa lúc task chạy chỉ áp dụng task kế), AC-510 (không nạp nửa file) | Khớp, không mâu thuẫn. AC-505..507, 509, 512 là rào chắn bổ sung (loại khóa quyền, chặn secret, symlink, ngân sách, cách ly giữa agent), không thuộc OWN-AC-1 |
| **AC-622** (OWN-AC-2) | AC-508 (agent ghi `.ai/skills/**` bị từ chối và audit), AC-513 (review-required, ghi `OVERLAY_MODIFIED_DURING_RUN`), AC-514 (merge bị từ chối nếu đổi `.ai/**`), AC-515 (Tier 2 không mount overlay) | AC-508 và AC-513 là phòng ngừa và phát hiện; AC-622 yêu cầu thêm hành vi tạm dừng Agent (design §12.1, §12.5: task sang `WAITING_APPROVAL`). AC-513 cần được hiểu theo §12.5. Xem OQ-D021-1 |
| **AC-623** (OWN-AC-3) | AC-511 (resume khi overlay đổi) | AC-511 "Then = ngữ cảnh bị làm mới khi lệch băm" (design §12.5). Cách làm mới (tắt snapshot hay phiên mới có tóm tắt) chờ PoC SK-03; phương án (b) chỉ gửi thông điệp cập nhật đã bị thay thế bởi §12.2 |

Ghi chú tác động lên AC đã có: AC-511 và AC-513 trong design được ghi chú, không viết lại; không AC APPROVED nào bị sửa.
