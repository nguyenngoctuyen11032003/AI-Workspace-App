# TRACEABILITY — PART 2 (FR-101 … FR-199)

**Trạng thái:** Draft Phase 0, chưa được Owner duyệt. Nguồn: `FUNCTIONAL_REQUIREMENTS.md`, `ACCEPTANCE_CRITERIA.md`, `AC_ADDENDUM_PART2.md`.
Ghi chú: file FR chỉ định nghĩa 62 FR trong dải 101…199 (không tồn tại FR-136…150, 166…180, 189…195); bảng dưới liệt kê đủ 62 FR, mỗi FR đúng 1 lần.
Status (OWR-3: **`AC_PROPOSED_OWNER_DECIDED` (giá trị mới, OWR-3) = có AC PROPOSED (agent viết câu chữ), Owner đã xác nhận quyết định ở OWR-3 nên không còn câu hỏi chặn; AC vẫn là PROPOSED, không phải APPROVED, vì agent không được tự chuyển PROPOSED thành APPROVED; `WONT_MVP` (giá trị mới, OWR-3) = FR bị Owner loại khỏi MVP (WON'T DO IN MVP), giữ hàng truy vết và không viết AC.** `AC_PROPOSED_PENDING_CLARIFICATION` (OWR-2) nay không còn FR nào.) — Status gốc: AC_DEFINED = đã có AC trong ACCEPTANCE_CRITERIA.md; AC_ADDED_PHASE0 = AC bổ sung ở PART2; NEEDS_OWNER_CLARIFICATION = THIẾU THÔNG TIN - cần Owner làm rõ (xem Q-xx trong AC_ADDENDUM_PART2.md). AC_PROPOSED_PENDING_CLARIFICATION (trạng thái mới, OWR-2) = đã có AC đề xuất PROPOSED (AC-601…AC-614) diễn đạt lại đề xuất của Owner, nhưng câu hỏi mở của Owner chưa được trả lời; chưa AC nào APPROVED.

| FR ID | Priority | Acceptance Criteria ID(s) | Test Scenario | Expected Result | Verification Status |
|---|---|---|---|---|---|
| FR-101 | M | AC-301 | Kill backend sau node 1 rồi mở lại | Trạng thái node 1 COMPLETED còn trong DB, 0 khác biệt | AC_ADDED_PHASE0 |
| FR-102 | M | AC-006, AC-110, AC-122 | Resume sau pause/stop/kill | Tiếp tục từ checkpoint, không chạy lại bước xong | AC_DEFINED |
| FR-103 | M | AC-057, AC-123 | Provider timeout tạm thời | Retry theo max retry/backoff, hiển thị retry count | AC_DEFINED |
| FR-104 | M | AC-302 | Chạy Condition với true rồi false | Chỉ nhánh được chọn thực thi | AC_ADDED_PHASE0 |
| FR-105 | M | AC-048, AC-303 | Parallel 3 nhánh rồi Merge | Merge chỉ chạy khi cả 3 nhánh xong | AC_ADDED_PHASE0 |
| FR-106 | M | AC-043 | Stage 2 hoàn thành | Task WAITING_APPROVAL, không sang Stage 3 | AC_DEFINED |
| FR-107 | M | AC-304 | Tạo task rồi đọc lại | Đủ 18 trường; thiếu title/project bị từ chối | AC_ADDED_PHASE0 |
| FR-108 | M | AC-060 | Đi qua vòng đời task | Chỉ 10 trạng thái hợp lệ, chuyển sai bị từ chối | AC_DEFINED |
| FR-109 | M | AC-055 | AUTO assignment cho task READY | Chọn agent điểm cao nhất, ghi lý do xem được **[Ghi chú OWR-2 Q-09: chấm điểm deterministic, không LLM chỉ để chấm điểm, PM chỉ can thiệp khi task mắc kẹt; AC-055 thêm ghi chú, không viết lại]** | AC_DEFINED |
| FR-110 | M | AC-056 | Owner assign thủ công | Gán mode MANUAL, ghi đè AUTO | AC_DEFINED |
| FR-111 | M | AC-007, AC-057 | Agent hết retry hoặc bị Disable | TASK_HANDOVER sang agent khác kèm context/log | AC_DEFINED |
| FR-112 | S | AC-305 | Kéo 2 node và nối cạnh (P2) | Node và cạnh hiển thị, lưu giữ sau reload | AC_ADDED_PHASE0 |
| FR-113 | S | AC-306 | Mở palette và thả từng loại (P2) | Đúng 9 UI node type (Agent, Task, Approval, Condition, Parallel, Merge, Tool, Test, Deployment), mỗi loại thả và lưu được; chỉ nói về UI node type **[Nguồn: OWR-3 A.14]** | AC_ADDED_PHASE0 |
| FR-114 | S | AC-061 | Tạo project từ template đã lưu | Workflow chạy đúng cấu trúc template | AC_DEFINED |
| FR-115 | C | AC-307 | Dry-run workflow (P2) | 0 lời gọi provider, hiển thị đường đi mô phỏng | AC_ADDED_PHASE0 |
| FR-116 | M | AC-058 | Kiểm event store | Mọi event có id, type, timestamp, actor, entity, payload, correlation id | AC_DEFINED |
| FR-117 | M | AC-308 | Kiểm registry và contract test event | Đúng 26 event type, mỗi type phát được event hợp lệ | AC_ADDED_PHASE0 |
| FR-118 | M | AC-059 | Tắt event bus tới UI | UI không cập nhật | AC_DEFINED |
| FR-119 | M | AC-041 | Tạo project chọn template SDLC | Workflow 8 giai đoạn khởi chạy theo thứ tự | AC_DEFINED |
| FR-120 | M | AC-041, AC-042 | Chạy Stage 1 | Project tạo, PM phân tích, đề xuất team và backlog | AC_DEFINED |
| FR-121 | M | AC-043 | Stage 2 BA hoàn thành | Approval gate, APPROVAL_REQUESTED ở Approval Center | AC_DEFINED |
| FR-122 | M | AC-601, AC-602, AC-603, AC-604 | Deliverable package của Workflow Run: required nodes, lưu và xem trong workspace | Artifact bắt buộc = output của node required; MVP lưu/xem trong workspace, không export ZIP. Còn mở không chặn: OQ-122-3 (phạm vi Stage 3 hay mọi run) **[Nguồn: OWR-3 A.3]** | AC_PROPOSED_OWNER_DECIDED |
| FR-123 | M | AC-047 | Stage 4 hoàn thành | Approval gate như AC-043 | AC_DEFINED |
| FR-124 | M | AC-048, AC-049 | Stage 5 chạy FE/BE/DB | Task song song trên branch riêng, merge sau review pass | AC_DEFINED |
| FR-125 | M | AC-050 | QA phát hiện lỗi ở Stage 6 | BUG_CREATED, bug liên kết task sửa lỗi | AC_DEFINED |
| FR-126 | M | AC-051 | Hoàn thành Stage 7 | PREVIEW_READY, không có bước deploy production | AC_DEFINED |
| FR-127 | M | AC-052, AC-053 | Owner Approve hoặc Reject ở Stage 8 | Project COMPLETED hoặc revision task tạo ra | AC_DEFINED |
| FR-128 | M | AC-053 | Reject với 2 feedback | 2 revision task targeted, không chạy lại Stage 1-4 | AC_DEFINED |
| FR-129 | M | AC-045, AC-053, AC-054 | Revision hoàn tất | Version +1, lịch sử xem được n và n+1 | AC_DEFINED |
| FR-130 | S | AC-053 | Reject kèm screenshot markup | Markup gắn vào feedback và revision task | AC_DEFINED |
| FR-131 | M | AC-041 | Tạo project từ prompt | Project có tên, mô tả, repo, workflow template | AC_DEFINED |
| FR-132 | M | AC-042 | Xem team đề xuất | Owner xem và sửa được team trước khi chạy | AC_DEFINED |
| FR-133 | M | AC-062, AC-309 | Pause/Resume và Cancel workflow | Pause/Resume từ checkpoint; Cancel đưa workflow và task về CANCELLED | AC_ADDED_PHASE0 |
| FR-134 | S | AC-310 | Mở Timeline workflow SDLC | Hiển thị giai đoạn, thời gian, agent, kết quả, approval khớp event store | AC_ADDED_PHASE0 |
| FR-135 | M | AC-045 | Reject approval gate Stage 2 | Revision task tạo, quay lại gate sau khi sửa | AC_DEFINED |
| FR-151 | M | AC-086 | Mở Virtual Office | 14 phòng đúng tên | AC_DEFINED |
| FR-152 | M | AC-087 | Zoom/pan bản đồ | Phóng/thu và kéo mượt, FPS ≥ 30 | AC_DEFINED |
| FR-153 | C | AC-644, AC-645 | Mở Office; thêm phòng ban/agent | Đúng 1 tầng, không điều khiển chuyển tầng, không tự mở rộng **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |
| FR-154 | C | AC-311 | Edit layout, di chuyển/đổi kích thước phòng (P2) | Vị trí và kích thước lưu qua reload | AC_ADDED_PHASE0 |
| FR-155 | C | AC-312 | Đặt và di chuyển furniture (P2) | Furniture ở ô đích, lưu qua reload | AC_ADDED_PHASE0 |
| FR-156 | M | AC-088 | AGENT_CREATED | Nhân vật chibi pixel-art xuất hiện | AC_DEFINED |
| FR-157 | M | AC-088 | Tạo agent ở phòng ban Frontend | Nhân vật tới desk trong phòng FE | AC_DEFINED |
| FR-158 | M | AC-089, AC-090, AC-092, AC-093, AC-095, AC-096 | Phát event qua harness cho 13 animation | Cả 13 animation render đúng | AC_DEFINED |
| FR-159 | M | AC-089 | TASK_STARTED | Speech bubble hiển thị tên task | AC_DEFINED |
| FR-160 | M | AC-089, AC-090, AC-091 | Không có event trong 5 phút | Không nhân vật nào di chuyển/đổi animation | AC_DEFINED |
| FR-161 | M | AC-094 | Click nhân vật | Popover có trạng thái, task, nút Chat/Assign/Pause/Resume/Profile hoạt động | AC_DEFINED |
| FR-162 | S | AC-097 | Sửa mục ánh xạ TASK_STARTED rồi phát event | Nhân vật chạy chuỗi mới, cấu hình lưu qua restart | AC_ADDED_PHASE0 |
| FR-163 | M | AC-098 | Tạo agent phòng ban mặc định và gán phòng ban tùy chỉnh | Desk nằm đúng phòng theo ánh xạ | AC_ADDED_PHASE0 |
| FR-164 | S | AC-099 | Phát/không phát event Idle | Chỉ agent có event Idle vào Relax/Coffee **[Ghi chú OWR-2 Q-14: mặc định idle không vào Coffee/Relax, setting tùy chọn; AC-099 thêm ghi chú]** | AC_ADDED_PHASE0 |
| FR-165 | S | AC-100 | Click phòng QA | Danh sách agent và task khớp DB; phòng rỗng hiển thị 0 | AC_ADDED_PHASE0 |
| FR-181 | M | AC-605, AC-606, AC-607 | Incident: trường bắt buộc, không suy diễn, grouping 15 phút có bộ đếm | Approval chờ lâu là operational alert, không phải incident; cùng type + target trong 15 phút gom, có occurrence_count **[Nguồn: OWR-3 A.4]** | AC_PROPOSED_OWNER_DECIDED |
| FR-182 | M | AC-608, AC-609, AC-610, AC-611 | Progress theo trọng số task, baseline, cancelled | Baseline = task ban đầu + thay đổi phạm vi đã duyệt; cancelled loại khỏi tử số và mẫu số; trọng số bằng nhau. Còn mở không chặn: OQ-182-4 (theo giai đoạn) **[Nguồn: OWR-3 A.5]** | AC_PROPOSED_OWNER_DECIDED |
| FR-183 | M | AC-612, AC-613, AC-614 | Dashboard nhiều metric tách biệt Progress | Nhiều metric, không KPI tổng hợp trong MVP **[Nguồn: OWR-3 A.6]** | AC_PROPOSED_OWNER_DECIDED |
| FR-184 | S | AC-313 | Mở Workflow dashboard | Liệt kê workflow chạy, giai đoạn hiện tại, thời gian chờ approval đúng | AC_ADDED_PHASE0 |
| FR-185 | M | AC-314 | Nhóm chi phí theo 4 chiều và thời gian | Tổng mỗi nhóm = tổng usage, sai số 0 | AC_ADDED_PHASE0 |
| FR-186 | S | AC-646, AC-647, AC-648 | Tập task có completed/failed/running/blocked/cancelled; mẫu số 0 | Success rate = completed/(completed+failed)×100%; loại running, blocked, cancelled; N/A khi mẫu số = 0. Còn mở không chặn: OQ-186-1 (test pass rate, reject rate, security findings) **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |
| FR-187 | S | AC-315 | Mở System dashboard | Hiển thị provider, sandbox, hàng đợi, concurrency khớp thực tế | AC_ADDED_PHASE0 |
| FR-188 | M | AC-316 | Mở dashboard ở workspace mới (bản release) | Chỉ giá trị 0/empty, không số mẫu; scan bundle 0 fixture | AC_ADDED_PHASE0 |
| FR-196 | M | AC-043 | Có yêu cầu approval | Approval Center liệt kê yêu cầu kèm loại, project, giai đoạn, agent, thời gian chờ, artifact | AC_DEFINED |
| FR-197 | M | AC-044, AC-045, AC-046 | Approve, Reject có và không có feedback | Approve/Reject ghi event; Reject thiếu feedback bị chặn | AC_DEFINED |
| FR-198 | M | AC-104, AC-317, AC-318 | Bật/tắt approval theo stage và loại hành động | Hành động/stage được cấu hình tạo approval và chỉ chạy sau approve | AC_ADDED_PHASE0 |
| FR-199 | M | AC-319, AC-320 | Cài mới; cố tắt approval production deploy | Stage 2, 4, 8 bắt buộc; production deploy không tắt được | AC_ADDED_PHASE0 |

## Thống kê

| Status | Số FR |
|---|---|
| AC_DEFINED | 34 |
| AC_ADDED_PHASE0 | 22 |
| AC_PROPOSED_OWNER_DECIDED (OWR-3: FR-122, 153, 181, 182, 183, 186) | 6 |
| AC_PROPOSED_PENDING_CLARIFICATION | 0 |
| NEEDS_OWNER_CLARIFICATION | 0 |
| **Tổng** | **62** |

## Mã AC đã bị tham chiếu nhưng chưa định nghĩa, nay được định nghĩa (code -> FR)

- AC-097 -> FR-162
- AC-098 -> FR-163
- AC-099 -> FR-164
- AC-100 -> FR-165

Giả định về ý định gốc (nguồn: `IMPLEMENTATION_ROADMAP.md` dòng 45, 338-350, "AC-086…100"). Các khoảng trống AC-065…070, 083…085, 115…120 chỉ xuất hiện trong khung "AC-001…AC-130", không có ngữ cảnh trích dẫn nên không được định nghĩa ở PART2.
