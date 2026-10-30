# PHASE_0_ACCEPTANCE_CRITERIA_TRACEABILITY.md

Ngày: 2026-10-30. Khắc phục QA P0-1. Trạng thái toàn bộ AC mới: **PROPOSED** (chưa được Owner duyệt). Không đổi phạm vi sản phẩm: AC chỉ diễn đạt lại điều FR đã yêu cầu.

**Chú giải trạng thái xác minh (Verification Status):** `AC_DEFINED` = đã có AC trong `ACCEPTANCE_CRITERIA.md`; `AC_ADDED_PHASE0` = AC mới PROPOSED do Phase 0 thêm; `NEEDS_OWNER_CLARIFICATION` = chưa đủ thông tin để viết AC; **`AC_PROPOSED_PENDING_CLARIFICATION` (giá trị mới, OWR-2) = đã có AC đề xuất (PROPOSED, không phải APPROVED) diễn đạt lại đề xuất của Owner, nhưng các câu hỏi mở của Owner chưa được trả lời nên AC chưa thể chốt.**

## 1. Kết quả kiểm tra bằng lệnh
| Chỉ số | Giá trị |
|---|---|
| FR định nghĩa trong `FUNCTIONAL_REQUIREMENTS.md` | 199 (mã chạy tới FR-292, không liên tục) |
| FR xuất hiện trong bảng truy vết | 199, mỗi FR đúng một dòng, không thiếu, không thừa |
| AC_DEFINED (đã có AC từ trước) | 109 |
| AC_ADDED_PHASE0 (AC mới, trạng thái PROPOSED) | 74 |
| AC_PROPOSED_PENDING_CLARIFICATION (OWR-2: có AC đề xuất PROPOSED, câu hỏi Owner còn mở) | 4 |
| NEEDS_OWNER_CLARIFICATION (chưa đủ thông tin, còn lại) | 12 |
| Tổng | 199 |

Phát hiện: (1) QA ghi 50 trên 199 FR thiếu AC, còn đợt này tìm thấy 90 FR chưa có AC (74 nay đã có AC mới, 16 chưa thể viết AC). Chênh lệch có thể do cách đối chiếu mã FR trong AC cũ khác nhau và chưa được đối soát chi tiết; (2) "30 mã AC được dẫn nhưng chưa định nghĩa" chủ yếu là khoảng trống đánh số trong cụm "AC-001..130". Chỉ AC-097..100 có ngữ cảnh trích dẫn và đã được định nghĩa với giả định về ý định gốc (xem phần 3). Các khoảng trống AC-016..020, 035..040, 065..070, 083..085, 115..120 được giữ nguyên là mã dự trữ, không bịa nội dung; `TESTING.md` và `IMPLEMENTATION_ROADMAP.md` cần cập nhật khi Owner chốt.

## 2. Mười sáu FR cần Owner làm rõ
Trong đó 9 FR mức Must hoặc Should chưa thể viết AC kiểm thử được: 4 Must (FR-122, FR-181, FR-182, FR-183) và 5 Should (FR-006, FR-009, FR-088, FR-186, FR-283). Bốn FR Must thuộc dashboard và workflow nên cần Owner trả lời sớm. Mã câu hỏi Q trong các bảng bên dưới chỉ duy nhất trong từng phần (Phần 1, 2, 3); khi tham chiếu dùng dạng "P1-Q78", "P2-Q01", "P3-Q01".

Không FR nào được tự gán AC bằng cách bịa tiêu chí. Mỗi FR dưới đây giữ trạng thái NEEDS_OWNER_CLARIFICATION cho tới khi Owner trả lời. **Cập nhật OWR-2:** bốn FR Must (FR-122, FR-181, FR-182, FR-183) nay có AC đề xuất và chuyển sang AC_PROPOSED_PENDING_CLARIFICATION (xem mục 5); còn 12 FR giữ NEEDS_OWNER_CLARIFICATION.

## 3. Bảng truy vết đầy đủ
Cột: FR ID, Priority, AC ID, Test Scenario, Expected Result, Verification Status. (Verification Status ở đây là trạng thái truy vết trong tài liệu, không phải kết quả kiểm thử đã chạy. Chưa AC nào được thực thi.)

### 3.1 Phần 1: FR-001..FR-100 (76 FR)
| FR ID | Priority | Acceptance Criteria ID(s) | Test Scenario | Expected Result | Verification Status |
|---|---|---|---|---|---|
| FR-001 | M | AC-201 | Tạo company có/không logo | Company lưu và hiển thị, tồn tại sau restart | AC_ADDED_PHASE0 |
| FR-002 | C | AC-202 | Chuyển giữa 2 workspace | Dữ liệu agent/skill/project cô lập theo workspace | AC_ADDED_PHASE0 |
| FR-003 | M | AC-203 | Tạo company mới | Đúng 12 phòng ban mặc định đúng tên | AC_ADDED_PHASE0 |
| FR-004 | M | AC-204 | CRUD phòng ban, xóa phòng còn agent | Xóa bị từ chối khi còn agent, thành công sau khi chuyển agent | AC_ADDED_PHASE0 |
| FR-005 | S | AC-205 | Chỉ định rồi đổi trưởng phòng | Mỗi phòng có đúng 1 trưởng phòng, hiển thị đúng | AC_ADDED_PHASE0 |
| FR-006 | S | (không có) | Hỏi Owner: số cấp quản lý và quy tắc reporting chain | Chưa xác định (Q-06) | NEEDS_OWNER_CLARIFICATION |
| FR-007 | S | AC-206 | Mở Org Chart | Cây 4 cấp đủ 12 phòng, mỗi agent đúng một lần | AC_ADDED_PHASE0 |
| FR-008 | S | AC-207 | Kéo-thả agent sang phòng/manager khác | Phòng và manager cập nhật, event được ghi | AC_ADDED_PHASE0 |
| FR-009 | S | (không có) | Hỏi Owner: đơn vị ngân sách, ngưỡng, hành vi khi vượt | Chưa xác định (Q-09) | NEEDS_OWNER_CLARIFICATION |
| FR-010 | S | AC-208 | Mở hiệu suất phòng ban có dữ liệu thật | 5 chỉ số khớp truy vấn DB độc lập | AC_ADDED_PHASE0 |
| FR-011 | M | AC-209 | Mở Virtual Office với 12 phòng ban mặc định | Mỗi phòng ban gắn đúng một phòng văn phòng | AC_ADDED_PHASE0 |
| FR-021 | M | AC-001 | Tạo agent qua Agent Builder đủ trường | Agent lưu, hiển thị, Created → Idle → Available | AC_DEFINED |
| FR-022 | M | AC-002, AC-003 | Name trống/trùng; mở dropdown model | Lỗi tại trường; chỉ model thuộc ListModels | AC_DEFINED |
| FR-023 | S | AC-210 | Chọn avatar có sẵn trong Agent Builder | Chỉ avatar có sẵn; hiển thị ở Management và Virtual Office | AC_ADDED_PHASE0 |
| FR-024 | M | AC-001 | Theo dõi vòng đời sau khi tạo agent | Chuyển trạng thái qua event theo lifecycle | AC_DEFINED |
| FR-025 | M | AC-001, AC-004 | Gán và bắt đầu task | Trạng thái đổi qua event, đồng nhất ≤ 500 ms | AC_DEFINED |
| FR-026 | M | AC-008 | Mở Agent Detail | Đủ 10 tab đúng tên | AC_DEFINED |
| FR-027 | M | AC-009 | Chat với agent từ profile | Streaming hiển thị, hội thoại lưu sau restart | AC_DEFINED |
| FR-028 | M | AC-211 | Nhấn Assign, chọn task READY hoặc tạo mới | Task được gán, chỉ task READY được liệt kê | AC_ADDED_PHASE0 |
| FR-029 | M | AC-005, AC-006 | Pause rồi Resume agent | Paused rồi tiếp tục từ checkpoint, không chạy lại bước xong | AC_DEFINED |
| FR-030 | M | AC-007 | Disable agent đang có task | Agent Disabled, task handover kèm log | AC_DEFINED |
| FR-031 | M | AC-014 | Xóa agent khi có task RUNNING | Từ chối xóa cho tới khi hết RUNNING | AC_DEFINED |
| FR-032 | M | AC-010 | Agent A gửi message cho B | B nhận qua message bus, message còn sau restart | AC_DEFINED |
| FR-033 | S | AC-212 | A bàn giao handoff document cho B | Tài liệu lưu, gắn task, xem được ở cả hai | AC_ADDED_PHASE0 |
| FR-034 | S | AC-213 | A gửi request help tới manager/agent khác | AGENT_HELP_REQUESTED được ghi, request xem được | AC_ADDED_PHASE0 |
| FR-035 | M | AC-013 | Agent tạo subtask | Subtask hiện trên Task Board có parent link | AC_DEFINED |
| FR-036 | S | AC-010 | Agent báo cáo cho reporting manager qua message | Manager nhận, hiển thị ở Conversations | AC_DEFINED |
| FR-037 | C | AC-214 | Agent đề xuất thay đổi ngoài phạm vi task | Đề xuất chờ duyệt, chỉ áp dụng sau approve | AC_ADDED_PHASE0 |
| FR-038 | C | (không có) | Hỏi Owner: meeting là cơ chế thật hay animation | Chưa xác định (Q-38) | NEEDS_OWNER_CLARIFICATION |
| FR-039 | M | AC-011 | Agent ghi file thuộc task khác | Bị chặn, audit ghi, agent nhận lỗi | AC_DEFINED |
| FR-040 | M | AC-009, AC-010, AC-126 | Restart sau hội thoại/message quan trọng | Conversations và message còn nguyên | AC_DEFINED |
| FR-041 | M | AC-012 | Agent chạy 3 task, mở tab Performance | Token/cost/lỗi/thời gian khớp GetUsage | AC_DEFINED |
| FR-042 | S | AC-215 | Bật/tắt memory, đổi phạm vi, xem và xóa | Phạm vi nạp đúng; nội dung xóa không còn trong prompt | AC_ADDED_PHASE0 |
| FR-043 | S | AC-015 | Tạo agent từ template vai trò BA | Form điền sẵn các trường theo template | AC_DEFINED |
| FR-044 | M | AC-004 | Đổi trạng thái agent, quan sát 4 màn hình | Trạng thái đồng nhất ≤ 500 ms | AC_DEFINED |
| FR-051 | M | AC-071 | Mở Skill Library mới | 6 category, mỗi category ≥ 1 skill ví dụ | AC_DEFINED |
| FR-052 | M | AC-216 | Tìm và lọc theo category, tag, agent | Kết quả đúng và đủ so với DB | AC_ADDED_PHASE0 |
| FR-053 | M | AC-072 | Tạo custom skill đủ trường | Skill lưu với đủ cấu trúc | AC_DEFINED |
| FR-054 | S | AC-072 | Upload 1 MD và 1 PDF làm references | `references/` chứa 2 file | AC_DEFINED |
| FR-055 | M | AC-072 | Kiểm tra thư mục skill | Có SKILL.md, references/, examples/, scripts/ | AC_DEFINED |
| FR-056 | S | AC-073 | Sửa skill v1 và lưu | Tạo v2, agent gán v1 không đổi | AC_DEFINED |
| FR-057 | S | AC-217 | Chạy test skill với prompt mẫu | Kết quả hiển thị, agent thật không bị đổi skill | AC_ADDED_PHASE0 |
| FR-058 | M | AC-074 | Nhập chỉ dẫn dạy agent | Nhận chỉ dẫn, chưa lưu gì | AC_DEFINED |
| FR-059 | M | AC-074 | Parse chỉ dẫn | Danh sách rule có cấu trúc hiển thị | AC_DEFINED |
| FR-060 | M | AC-074 | Sau parse | Gợi ý tạo/cập nhật skill | AC_DEFINED |
| FR-061 | M | AC-074, AC-075, AC-076 | Sửa/xóa rule; đóng màn hình trước khi Lưu | Lưu đúng sau chỉnh sửa; đóng thì không persist | AC_DEFINED |
| FR-062 | M | AC-077 | Gán một skill cho 3 agent | 3 agent có skill; skill thấy 3 agent dùng | AC_DEFINED |
| FR-063 | M | AC-078 | Verify agent đã load skill | Agent tóm tắt rule; chỉ pass mới "đã load" | AC_DEFINED |
| FR-064 | S | AC-079 | Chạy evaluation | Kết quả pass/fail từng test | AC_DEFINED |
| FR-065 | M | AC-080 | Task Testing với agent có 10 skill | Chỉ skill liên quan được nạp | AC_DEFINED |
| FR-066 | S | AC-081 | Agent chạy task có skill mandatory rule | Prompt chứa rule đánh dấu bắt buộc | AC_DEFINED |
| FR-067 | S | AC-218 | Gọi tool ngoài allowed tools khi áp dụng skill | Tool ngoài danh sách bị chặn, audit ghi | AC_ADDED_PHASE0 |
| FR-068 | M | AC-082 | Gỡ skill ở tab Knowledge | Task tiếp theo không nạp skill đó | AC_DEFINED |
| FR-076 | M | AC-021 | Connect Claude Code đã login | Connected, ListModels ≥ 1, PROVIDER_CONNECTED | AC_DEFINED |
| FR-077 | M | AC-022 | Connect Codex đã login | Connected như Claude (UNVERIFIED nếu chưa cài Codex) **[Ghi chú OWR-2 D-005: FR hoãn khỏi MVP; AC-022 chưa áp dụng cho MVP, chờ Owner xác nhận điều chỉnh nhóm B]** | AC_DEFINED |
| FR-078 | C | (không có) | Hỏi Owner: cơ chế tích hợp OpenHands | Chưa xác định (Q-78) **[Ghi chú OWR-2 D-006/Q-02: hoãn đến sau Phase 4]** | NEEDS_OWNER_CLARIFICATION |
| FR-079 | S | AC-219 | Adapter generic CLI với binary giả | Lệnh đúng tham số; output thành đúng event chuẩn | AC_ADDED_PHASE0 |
| FR-080 | M | AC-021, AC-022 | Connect bằng CLI đã login, không có API key | Kết nối thành công, không yêu cầu API key | AC_DEFINED |
| FR-081 | M | AC-021, AC-025, AC-034 | Gọi operation hỗ trợ và không hỗ trợ | Hoạt động đúng; không hỗ trợ trả NOT_SUPPORTED | AC_DEFINED |
| FR-082 | M | AC-021, AC-023 | Luồng phát hiện → login → hướng dẫn → kiểm tra lại | Hướng dẫn từng bước, kiểm tra lại hoạt động | AC_DEFINED |
| FR-083 | M | AC-023, AC-024 | CLI chưa cài / chưa login | Trạng thái Not installed / Not authenticated | AC_DEFINED |
| FR-084 | M | AC-024 | Kiểm tra UI khi chưa login | Không có ô nhập mật khẩu | AC_DEFINED |
| FR-085 | M | AC-031 | Rà soát mã adapter | Không có mã đọc file credential của CLI | AC_DEFINED |
| FR-086 | M | AC-029, AC-113 | Quét DB/log bằng scanner secret | 0 credential plaintext | AC_DEFINED |
| FR-087 | M | AC-030 | Bắt payload IPC/HTTP tới frontend | Không payload nào chứa credential | AC_DEFINED |
| FR-088 | S | (không có) | Hỏi Owner: có lưu credential, shell Tauri/web | Chưa xác định (Q-88) | NEEDS_OWNER_CLARIFICATION |
| FR-089 | M | AC-027, AC-028 | Disconnect/Connect lại; phiên hết hạn | Kết nối lại được; SESSION_EXPIRED kèm hướng dẫn | AC_DEFINED |
| FR-090 | M | AC-029, AC-113 | Quét log sau 1 ngày chạy | Không credential trong log | AC_DEFINED |
| FR-091 | M | AC-032 | Kiểm tra sandbox hai agent cùng provider | Không credential; gọi provider qua adapter | AC_DEFINED |
| FR-092 | M | AC-033 | Mở Provider Hub | Hiển thị trạng thái, model, usage, lỗi, số session (dữ liệu thật) | AC_DEFINED |
| FR-093 | M | AC-003 | Mở dropdown model | Chỉ model thuộc ListModels | AC_DEFINED |
| FR-094 | M | AC-025 | SendMessage qua provider | StreamEvents phát event chuẩn realtime | AC_DEFINED |
| FR-095 | M | AC-026 | CancelTask khi session chạy | CLI dừng ≤ 5 s | AC_DEFINED |
| FR-096 | S | AC-012 | Mở Performance khi CLI không cấp usage | Usage ước lượng được đánh dấu "ước lượng" | AC_DEFINED |
| FR-097 | S | AC-034 | ResumeSession khi provider không hỗ trợ | NOT_SUPPORTED và fallback tạo session mới | AC_DEFINED |
| FR-098 | C | (không có) | Hỏi Owner: có làm API key fallback không | Chưa xác định (Q-98) | NEEDS_OWNER_CLARIFICATION |

## Tổng kết

| Verification Status | Số FR |
|---|---|
| AC_DEFINED | 51 |
| AC_ADDED_PHASE0 | 19 |
| NEEDS_OWNER_CLARIFICATION | 6 |
| **Tổng** | **76** |

Số FR duy nhất trong dải FR-001…FR-100 = 76 (không phải 100; các số FR-012…020, 045…050, 069…075, 099, 100 chưa được định nghĩa trong FUNCTIONAL_REQUIREMENTS.md).

## Mã AC chưa định nghĩa đã xử lý (code → FR)

Không có mã AC nào trong dải AC-001…AC-065 được trích dẫn như tiêu chí cụ thể mà chưa định nghĩa. AC-016…020, 035…040, 065 chỉ xuất hiện như mô tả khoảng trống (CF-037, TESTING.md mục 7, ROADMAP rủi ro 5) và được giữ là số dự trữ, không bịa định nghĩa. Danh sách giải quyết: (rỗng).

## Câu hỏi cần Owner làm rõ (6)

FR-006 (Q-06), FR-009 (Q-09), FR-038 (Q-38), FR-078 (Q-78), FR-088 (Q-88), FR-098 (Q-98). Chi tiết ở `AC_ADDENDUM_PART1.md` mục 4. Q-11 (loại event cho FR-008) là câu hỏi bổ sung không chặn AC.

### 3.2 Phần 2: FR-101..FR-199 (62 FR)
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
| FR-113 | S | AC-306 | Mở palette và thả từng loại (P2) | Đúng 8 loại node, mỗi loại thả và lưu được **[Ghi chú OWR-2 Q-11: 9 UI node type + 11 engine kind; AC-306 (8 loại) cần đồng bộ, chờ Owner cung cấp danh sách]** | AC_ADDED_PHASE0 |
| FR-114 | S | AC-061 | Tạo project từ template đã lưu | Workflow chạy đúng cấu trúc template | AC_DEFINED |
| FR-115 | C | AC-307 | Dry-run workflow (P2) | 0 lời gọi provider, hiển thị đường đi mô phỏng | AC_ADDED_PHASE0 |
| FR-116 | M | AC-058 | Kiểm event store | Mọi event có id, type, timestamp, actor, entity, payload, correlation id | AC_DEFINED |
| FR-117 | M | AC-308 | Kiểm registry và contract test event | Đúng 26 event type, mỗi type phát được event hợp lệ | AC_ADDED_PHASE0 |
| FR-118 | M | AC-059 | Tắt event bus tới UI | UI không cập nhật | AC_DEFINED |
| FR-119 | M | AC-041 | Tạo project chọn template SDLC | Workflow 8 giai đoạn khởi chạy theo thứ tự | AC_DEFINED |
| FR-120 | M | AC-041, AC-042 | Chạy Stage 1 | Project tạo, PM phân tích, đề xuất team và backlog | AC_DEFINED |
| FR-121 | M | AC-043 | Stage 2 BA hoàn thành | Approval gate, APPROVAL_REQUESTED ở Approval Center | AC_DEFINED |
| FR-122 | M | AC-601, AC-602, AC-603, AC-604 | Deliverable package của Workflow Run (đề xuất Owner OWR-2 §4) | AC đề xuất; OQ-122-1/2/3 chờ Owner (Q-01) | AC_PROPOSED_PENDING_CLARIFICATION |
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
| FR-153 | C | (chưa có) | Chưa xác định số tầng và tiêu chí | Cần Owner làm rõ (Q-02) | NEEDS_OWNER_CLARIFICATION |
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
| FR-181 | M | AC-605, AC-606, AC-607 | Incident: trường bắt buộc, không suy diễn, grouping (đề xuất Owner OWR-2 §4) | AC đề xuất; OQ-181-1/2 chờ Owner (Q-03) | AC_PROPOSED_PENDING_CLARIFICATION |
| FR-182 | M | AC-608, AC-609, AC-610, AC-611 | Công thức Progress theo trọng số task (đề xuất Owner OWR-2 §4) | AC đề xuất; OQ-182-1/2/3/4 chờ Owner (Q-04) | AC_PROPOSED_PENDING_CLARIFICATION |
| FR-183 | M | AC-612, AC-613, AC-614 | Performance Metrics tách biệt Progress (đề xuất Owner OWR-2 §4) | AC đề xuất; OQ-183-1/2 chờ Owner (Q-05) | AC_PROPOSED_PENDING_CLARIFICATION |
| FR-184 | S | AC-313 | Mở Workflow dashboard | Liệt kê workflow chạy, giai đoạn hiện tại, thời gian chờ approval đúng | AC_ADDED_PHASE0 |
| FR-185 | M | AC-314 | Nhóm chi phí theo 4 chiều và thời gian | Tổng mỗi nhóm = tổng usage, sai số 0 | AC_ADDED_PHASE0 |
| FR-186 | S | (chưa có) | Chưa xác định công thức các rate | Cần Owner làm rõ (Q-06) | NEEDS_OWNER_CLARIFICATION |
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
| AC_PROPOSED_PENDING_CLARIFICATION | 4 |
| NEEDS_OWNER_CLARIFICATION | 2 |
| **Tổng** | **62** |

## Mã AC đã bị tham chiếu nhưng chưa định nghĩa, nay được định nghĩa (code -> FR)

- AC-097 -> FR-162
- AC-098 -> FR-163
- AC-099 -> FR-164
- AC-100 -> FR-165

Giả định về ý định gốc (nguồn: `IMPLEMENTATION_ROADMAP.md` dòng 45, 338-350, "AC-086…100"). Các khoảng trống AC-065…070, 083…085, 115…120 chỉ xuất hiện trong khung "AC-001…AC-130", không có ngữ cảnh trích dẫn nên không được định nghĩa ở PART2.

### 3.3 Phần 3: FR-200..FR-292 (61 FR)
| FR ID | Priority | Acceptance Criteria ID(s) | Test Scenario | Expected Result | Verification Status |
|---|---|---|---|---|---|
| FR-200 | C | — (Q-01) | Approval chờ quá N giờ | Chưa xác định: thiếu N, người nhận, hành động escalation | NEEDS_OWNER_CLARIFICATION |
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
| FR-283 | S | — (Q-02) | Kiểm giao diện "premium" | Chưa xác định: thiếu tiêu chí đo | NEEDS_OWNER_CLARIFICATION |
| FR-284 | C | — (Q-03) | Chuyển ngôn ngữ Việt/Anh | Chưa xác định: OI-07 chưa chốt | NEEDS_OWNER_CLARIFICATION |
| FR-286 | M | AC-121, AC-125, AC-126 | Kill app rồi mở lại | Dashboard, approval, hội thoại, message còn nguyên | AC_DEFINED |
| FR-287 | M | AC-121, AC-122 | Kill app, chọn Tiếp tục | Resume từ checkpoint, không chạy lại bước xong | AC_DEFINED |
| FR-288 | M | AC-123 | Provider timeout tạm thời | Retry theo max retry/backoff, hiển thị retry count | AC_DEFINED |
| FR-289 | M | AC-122 | Resume sau restart | Resume từ checkpoint gần nhất, không tạo trùng | AC_DEFINED |
| FR-290 | M | AC-124 | Kill tiến trình CLI từ ngoài | Phát hiện ≤ 60 s; task FAILED hoặc retry | AC_DEFINED |
| FR-291 | S | AC-122 | Resume step đã hoàn tất | Artifact không bị tạo trùng | AC_DEFINED |
| FR-292 | C | — (Q-04) | Sao lưu/xuất dữ liệu workspace | Chưa xác định: thiếu phạm vi và định dạng | NEEDS_OWNER_CLARIFICATION |

## Tổng kết

| Status | Số FR |
|---|---|
| AC_DEFINED | 24 |
| AC_ADDED_PHASE0 | 33 |
| NEEDS_OWNER_CLARIFICATION | 4 |
| **Tổng** | **61** |

## 4. Nội dung AC mới (PROPOSED)

### 4.1 AC-201..AC-219 và các AC khôi phục (Phần 1)
# ACCEPTANCE CRITERIA ADDENDUM — PART 1 (FR-001 … FR-100) — AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0 finalization) · **Vai trò:** BA · **Trạng thái:** DRAFT — chưa có hạng mục nào được Owner phê duyệt · **Ngày:** 2026-10-30
**Liên quan:** QA_REVIEW_PHASE0 P0-1, CF-037. File này **không sửa** `ACCEPTANCE_CRITERIA.md`; chỉ bổ sung. Khi Owner duyệt, nội dung được hợp nhất vào file gốc.

> Quy ước: cùng định dạng Given/When/Then như `ACCEPTANCE_CRITERIA.md`, thêm cột Verification và Priority (sao chép từ FR). AC chỉ diễn đạt lại điều FR đã yêu cầu, không thêm tính năng. Giá trị số ghi "Đề xuất" cần Owner chốt. Dải mã dùng: AC-201 … AC-219 (dải AC-201…AC-299 dành cho Part 1).

## 1. Ghi chú phạm vi thực tế

- `FUNCTIONAL_REQUIREMENTS.md` chỉ định nghĩa **76 FR** trong dải FR-001…FR-100 (FR-012…020, 045…050, 069…075, 099, 100 là số để trống). Bảng truy vết Part 1 liệt kê đúng 76 FR đó.
- 51 FR đã có AC trong `ACCEPTANCE_CRITERIA.md`; 25 FR chưa có AC → 19 FR được viết AC mới (AC-201…AC-219), 6 FR đánh dấu **THIẾU THÔNG TIN - cần Owner làm rõ** (mục 4).

## 2. AC mới bổ sung (AC_ADDED_PHASE0)

| ID | FR | Given | When | Then | Verification | Pri |
|---|---|---|---|---|---|---|
| AC-201 | FR-001 | App mở, chưa có company | Owner tạo company với tên "Acme", mô tả "Test", có và không có logo (hai lần chạy) | Cả hai lần company được lưu và hiển thị đúng tên/mô tả trên màn hình Company; khi không upload logo, company vẫn tạo được; sau restart app dữ liệu còn nguyên | E2E (Playwright) + integration test API `POST /companies` | M |
| AC-202 | FR-002 | Hai workspace W1 và W2, W1 có agent "A1", skill "S1" và project "P1" | Owner chuyển từ W1 sang W2 bằng bộ chuyển workspace | Danh sách agent/skill/project của W2 không chứa A1, S1, P1; gọi API với id của A1 trong ngữ cảnh W2 trả lỗi not-found/forbidden; chuyển lại W1 thì dữ liệu hiển thị đủ | Integration test (data isolation theo workspace) + E2E | C |
| AC-203 | FR-003 | Workspace sạch | Owner tạo company mới | Đúng 12 phòng ban được tạo tự động với tên: Management, Business Analysis, System Architecture, UI/UX Design, Frontend, Backend, Database, QA, Cybersecurity, DevOps, Code Review, Product Evaluation; không phòng ban nào thừa/thiếu/trùng tên | Integration test (đếm và so tên) + E2E | M |
| AC-204 | FR-004 | Company có phòng ban "Mobile" đang chứa 1 agent | (a) Owner tạo phòng "SEO" với tên, mô tả, màu, icon; (b) sửa màu của "SEO"; (c) xóa "SEO"; (d) xóa "Mobile" khi còn agent; (e) chuyển agent đi rồi xóa "Mobile" | (a)(b)(c) thành công và phản ánh trên UI sau reload; (d) bị từ chối với thông báo nêu số agent còn lại, phòng không bị xóa; (e) xóa thành công | Integration test + E2E | M |
| AC-205 | FR-005 | Phòng ban Backend có agent A và B | Owner chỉ định A làm trưởng phòng, sau đó đổi sang B | Phòng chỉ có đúng 1 trưởng phòng tại mọi thời điểm; Org Chart và màn hình Department hiển thị trưởng phòng hiện tại; sau khi đổi, A không còn là trưởng phòng; không chọn được agent thuộc phòng ban khác | Integration test + E2E | S |
| AC-206 | FR-007 | Company có 12 phòng ban, mỗi phòng có 1 trưởng phòng và ≥ 1 agent | Owner mở màn hình Org Chart | Cây hiển thị 4 cấp: company → phòng ban → trưởng phòng → agent; số node phòng ban = 12; mỗi agent xuất hiện đúng một lần dưới phòng ban của nó | E2E (đếm node, so với dữ liệu DB) | S |
| AC-207 | FR-008 (P2) | Org Chart; agent A thuộc phòng Frontend, manager M1 | Owner kéo-thả A sang phòng Backend, rồi sang manager M2 | A thuộc phòng Backend và reporting manager = M2 sau reload; một event được ghi vào event store với actor = Owner và entity = A (loại event cụ thể chưa có trong danh mục FR-117 — xem mục 4 Q-11); thao tác không hợp lệ (thả vào phòng không tồn tại) bị từ chối và dữ liệu không đổi | E2E + integration test | S |
| AC-208 | FR-010 (P2) | Phòng ban QA đã hoàn thành 4 task, thất bại 1 task, có 2 task bị reject, có dữ liệu cost từ GetUsage | Owner mở màn hình hiệu suất phòng ban QA | Hiển thị: số task hoàn thành = 4, thất bại = 1, thời gian trung bình = trung bình thực của các task hoàn thành, chi phí = tổng cost của các session, tỷ lệ reject = 2/5 (hoặc công thức Owner chốt); mọi số khớp truy vấn DB độc lập | Integration test (so với truy vấn SQL) + E2E | S |
| AC-209 | FR-011 | Company mới với 12 phòng ban mặc định | Owner mở Virtual Office | Mỗi phòng ban hiển thị gắn với đúng một phòng của Virtual Office theo bảng ánh xạ FR-163; không phòng ban nào không có phòng | E2E + unit test bảng ánh xạ | M |
| AC-210 | FR-023 | Màn hình Agent Builder | Owner mở bộ chọn avatar, chọn một sprite pixel-art và lưu agent | Chỉ các avatar thuộc bộ có sẵn được liệt kê (không có upload tự do); avatar đã chọn hiển thị trong Agent Management và cho nhân vật tương ứng trong Virtual Office; sau reload avatar không đổi | E2E | S |
| AC-211 | FR-028 | Agent A Available; có task T1 ở trạng thái READY và T2 ở BACKLOG | Owner nhấn Assign trên profile A (a) chọn T1, (b) tạo task mới T3 từ hộp thoại Assign | (a) T1 → ASSIGNED, assignee = A, mode MANUAL; (b) T3 được tạo và gán cho A; danh sách chọn chỉ chứa task READY (không chứa T2); event TASK_ASSIGNED được ghi cho mỗi lần gán | Integration test + E2E | M |
| AC-212 | FR-033 | Agent A đang làm task T1; agent B tồn tại | A bàn giao handoff document cho B | Tài liệu được lưu, gắn với T1 và xem được từ T1 và từ Conversations của cả A và B; nội dung còn nguyên sau restart; không gắn được vào task khác | Integration test + E2E | S |
| AC-213 | FR-034 | Agent A Running, có reporting manager M | A gửi request help tới M, và một request khác tới agent B | Event AGENT_HELP_REQUESTED được ghi cho mỗi request (actor = A, entity = đích); request lưu và xem được ở Conversations của người gửi và người nhận | Integration test | S |
| AC-214 | FR-037 (P2) | Agent A có task T1 (phạm vi file src/a/**) | A gửi propose change ảnh hưởng src/b/** (ngoài task của A) | Đề xuất được tạo ở trạng thái chờ duyệt, xuất hiện ở hàng chờ của manager/Owner; thay đổi **không** được áp dụng cho tới khi manager hoặc Owner approve; sau approve/reject trạng thái đề xuất cập nhật và còn trong lịch sử | Integration test + E2E | C |
| AC-215 | FR-042 | Agent A có memory đã ghi nội dung ở project P1 | Owner (a) tắt memory, (b) bật lại với phạm vi per-project, (c) xem nội dung, (d) xóa nội dung | (a) ở task tiếp theo không có memory nào được nạp hoặc ghi; (b) memory chỉ nạp khi chạy task thuộc P1, không nạp ở P2; (c) nội dung hiển thị đúng; (d) sau xóa, danh sách rỗng và task tiếp theo không còn dữ liệu đã xóa trong prompt (kiểm tra qua log có mask); phạm vi global nạp ở cả P1 và P2 | Integration test + E2E | S |
| AC-216 | FR-052 | Skill Library có ≥ 12 skill gồm nhiều category/tag; skill X được gán cho agent A, skill Y không gán cho agent nào | Owner (a) tìm theo từ khóa tên, (b) lọc theo category Testing, (c) lọc theo tag, (d) lọc theo agent A | Mỗi phép lọc trả đúng và đủ tập skill khớp điều kiện (so với DB), (d) chứa X và không chứa Y; kết hợp nhiều bộ lọc = giao của các tập; không có kết quả hiển thị trạng thái rỗng | Integration test (API search/filter) + E2E | M |
| AC-217 | FR-057 (P2) | Skill S và một agent thử nghiệm | Owner chạy "test skill" với prompt mẫu | Skill được gán tạm cho agent thử nghiệm, agent chạy prompt mẫu qua adapter thật, kết quả hiển thị cho Owner; agent thật của người dùng không bị đổi danh sách skill; phiên thử nghiệm được ghi vào Activity Logs | Integration test + E2E (provider thật, VERIFIED-REAL) | S |
| AC-218 | FR-067 | Skill S có allowed tools = {Read, Grep}; agent A (có thêm tool Write) áp dụng S cho task T | A cố gọi tool Write trong lúc áp dụng S, rồi gọi Read | Lời gọi Write bị chặn, audit log ghi, agent nhận thông báo lỗi; Read thực thi bình thường; sau khi gỡ S, Write dùng được lại (theo permission của agent) | Integration test (hook/canUseTool) + security test | S |
| AC-219 | FR-079 (P2) | Cấu hình adapter generic CLI trỏ tới binary giả có lệnh, tham số và parser output đã khai báo; binary in 3 dòng output cố định | Owner Connect rồi SendMessage qua adapter generic CLI | Lệnh được gọi đúng tham số đã cấu hình; output được parser chuyển thành đúng 3 event chuẩn; lệnh/tham số sai cho trạng thái lỗi rõ ràng, không treo | Integration test (binary giả, VERIFIED-MOCK) | S |

## 3. AC đã gọi tên nhưng chưa được định nghĩa trong dải AC-001…AC-065 (đã xử lý)

Đối chiếu mọi `AC-0xx` trong `TESTING.md`, `IMPLEMENTATION_ROADMAP.md`, `DEVELOPMENT_GUIDE.md`, `CODING_STANDARDS.md`, `COMPONENT_SPECIFICATION.md`, `CONFLICTS_TO_RESOLVE.md`:

- Tất cả mã AC được trích dẫn như tiêu chí cụ thể (AC-001…015, 021…034, 041…064) **đều đã có định nghĩa** trong `ACCEPTANCE_CRITERIA.md`.
- Các mã chưa định nghĩa trong dải này (AC-016…020, 035…040, 065) chỉ xuất hiện như **mô tả khoảng trống** (CONFLICTS_TO_RESOLVE CF-037; TESTING.md mục 7; ROADMAP bảng rủi ro dòng 5; AC-016 chỉ xuất hiện trong cụm "AC-016…020"), không được trích như một tiêu chí nghiệm thu cụ thể.
- Vì ý định gốc của các số này không xác định được (không có FR hay ngữ cảnh gắn kèm) nên **không bịa định nghĩa**; chúng tiếp tục là số dự trữ. Danh sách mã được giải quyết: **không có** (0).
- Lưu ý: một số trích dẫn dạng dải (vd. "AC-001…AC-130" trong TESTING.md dòng 11, "AC-043…046, 055…064" trong ROADMAP) ngầm bao gồm ID dự trữ; việc sửa văn bản các dải này thuộc phạm vi QA P0-1 phần (b), nằm ngoài hai file đầu ra của Part 1.

## 4. Cần Owner làm rõ (không đủ thông tin để viết AC kiểm thử được)

| # | FR | Câu hỏi cụ thể | Lý do không viết AC |
|---|---|---|---|
| Q-06 | FR-006 (S, P2) | Có bao nhiêu cấp quản lý chính thức và tên từng cấp là gì (FR chỉ nêu "ví dụ" Director → Department Head → Lead → Member)? Quy tắc reporting chain cụ thể (một agent có đúng một manager? giới hạn độ sâu?) | Số cấp là [Giả định], không có tiêu chí đúng/sai |
| Q-09 | FR-009 (S, P2) | Ngân sách phòng ban tính bằng token hay tiền (OI-FR-07)? Ngưỡng cảnh báo cuối cùng là 80%/100% hay khác? Khi đạt 100% hệ thống chỉ cảnh báo hay chặn thêm task? Cảnh báo hiển thị ở đâu (banner, thông báo, event nào)? | Đơn vị, ngưỡng và hành vi khi vượt chưa chốt; Then không đo được |
| Q-38 | FR-038 (C, P2) | Meeting là cơ chế thật (nhiều agent trao đổi qua message bus theo lượt, có biên bản do ai tạo) hay chỉ là animation (OI-FR-06, OI-PRD-07)? Biên bản gồm trường nào? Điều kiện kết thúc meeting? | Cơ chế chưa xác định |
| Q-78 | FR-078 (C, P2) | OpenHands tích hợp theo cơ chế nào (CLI, REST API, SDK)? 13 operation nào hỗ trợ và nghiệm thu bằng kịch bản nào? | FR ghi "cơ chế chưa rõ" |
| Q-88 | FR-088 (S) | Có chấp nhận lưu credential (API key fallback) trong MVP không (phụ thuộc FR-098/OI-06)? Shell là Tauri hay web (OI-FR-03) vì Windows Credential Manager chỉ truy cập được qua backend/shell native? Nếu không lưu credential nào thì FR này đánh dấu N/A? | Điều kiện áp dụng chưa chốt, chưa có kịch bản kiểm thử xác định |
| Q-98 | FR-098 (C, phase chưa chốt) | Có làm API key fallback không, ở phase nào (OI-06/OI-FR-04)? Nếu có: cách nhập key, nơi lưu, provider nào hỗ trợ, và xung đột với FR-084 (không nhập mật khẩu) được xử lý ra sao? | Chưa có quyết định, FR chưa có phạm vi |

Số câu hỏi: **6** (Q-11 nêu trong AC-207 là chú thích bổ sung về loại event của FR-008, không làm FR-008 mất AC: gộp vào câu hỏi AP-FR-03 danh mục event; xem Q-11 bên dưới).

| # | FR | Câu hỏi (không chặn AC) |
|---|---|---|
| Q-11 | FR-008 | Thao tác kéo-thả đổi phòng/manager phát sinh loại event nào trong danh mục FR-117 (AGENT_UPDATED hay loại mới)? AC-207 hiện chỉ yêu cầu "một event được ghi". |

## 5. Rủi ro / giả định

- AC-202 phụ thuộc OI-FR-10 (multi-company trong MVP hay P2); FR-002 là Could/P2.
- AC-208 dùng công thức tỷ lệ reject = số task bị reject / tổng task kết thúc là diễn giải của BA, cần Owner chốt.
- AC-211: mode MANUAL theo FR-110 (đã có AC-056).
- AC-217, AC-219 phụ thuộc phase P2; chưa thuộc phạm vi nghiệm thu MVP.

## 6. Cần người dùng phê duyệt

| # | Hạng mục |
|---|---|
| AP-AD1-01 | 19 AC mới AC-201…AC-219 |
| AP-AD1-02 | Trả lời 6 câu hỏi Q-06, Q-09, Q-38, Q-78, Q-88, Q-98 (và Q-11) |
| AP-AD1-03 | Giữ AC-016…020, 035…040, 065 là số dự trữ |

### 4.2 AC-301..AC-320, AC-097..100 (Phần 2)
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
- Câu hỏi cần Owner làm rõ: 7 (Q-00…Q-06); 6 FR bị gắn NEEDS_OWNER_CLARIFICATION (FR-122, 153, 181, 182, 183, 186; từ OWR-2: FR-122, 181, 182, 183 chuyển sang AC_PROPOSED_PENDING_CLARIFICATION, còn FR-153, FR-186 giữ NEEDS_OWNER_CLARIFICATION).
- Tổng hợp mã AC đã định nghĩa theo mã cũ: AC-097 → FR-162; AC-098 → FR-163; AC-099 → FR-164; AC-100 → FR-165.

### 4.3 AC-401..AC-433 (Phần 3)
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

## 5. Cập nhật OWR-2 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Phase 0 vẫn mở; Phase 1 chưa được Owner cho phép. **Không AC nào trong mục này là APPROVED**; tất cả là PROPOSED. Định nghĩa đầy đủ AC-601..AC-614 và AC-621..AC-623 nằm ở `docs/product/traceability_parts/AC_ADDENDUM_PART4_OWR2.md` (định nghĩa một lần duy nhất, không sao chép ở đây).

### 5.1 Nhật ký thay đổi

| # | Thay đổi | Nguồn | File |
|---|---|---|---|
| 1 | FR-122, FR-181, FR-182, FR-183: trích văn bản FR thật, thêm AC-601..AC-614 (PROPOSED), giữ các câu hỏi Owner thành OQ-122-1..3, OQ-181-1..2, OQ-182-1..4, OQ-183-1..2 (PENDING_OWNER_CLARIFICATION). Ghi nhận tên Owner dùng khác tên FR gốc | OWR-2 §4 | FUNCTIONAL_REQUIREMENTS.md mục 9.1; AC_ADDENDUM_PART4_OWR2.md |
| 2 | Trạng thái bốn FR trên: NEEDS_OWNER_CLARIFICATION thành AC_PROPOSED_PENDING_CLARIFICATION (giá trị mới, định nghĩa ở chú giải đầu file) | OWR-2 §4, §7 | file này; TRACEABILITY_PART2.md |
| 3 | AC-621..AC-623 (OWN-AC-1..3) gắn với FR-293..FR-298 đề xuất; PoC SK-01..SK-05 (chưa chạy); thêm FR-293..298 vào FR file dạng "đề xuất, chờ Owner duyệt", không tính vào 199 | OWR-2 Q-18, §5.5 | FUNCTIONAL_REQUIREMENTS.md mục 3.13; AC_ADDENDUM_PART4_OWR2.md |
| 4 | Đối chiếu AC-621..623 với AC-501..AC-515 của `DYNAMIC_SKILL_INJECTION.md` (không đánh số lại) | OWR-2 Q-18 | AC_ADDENDUM_PART4_OWR2.md mục 3 |
| 5 | FR-109: công thức deterministic, không LLM chỉ để chấm điểm; PM chỉ can thiệp khi task mắc kẹt | OWR-2 Q-09 | FR file; ghi chú ở bảng truy vết; AC-055 chỉ thêm ghi chú |
| 6 | FR-113: 9 UI node type và 11 engine kind (tên cụ thể PENDING); AC-306 thêm ghi chú | OWR-2 Q-11 | FR file; AC_ADDENDUM_PART2.md mục 5 |
| 7 | FR-164: mặc định idle không vào Coffee/Relax, setting tùy chọn; AC-099 thêm ghi chú | OWR-2 Q-14 | FR file; AC_ADDENDUM_PART2.md mục 5 |
| 8 | FR-077 (Codex) hoãn khỏi MVP, FR-078 (OpenHands) hoãn đến sau Phase 4; giữ FR-081 (AgentAdapter). Ghi nhận **xung đột với nhóm B** của tiêu chí nghiệm thu MVP gốc (AC-021, AC-022), cần Owner xác nhận; AC-022 chỉ thêm ghi chú | OWR-2 D-005, D-006, Q-02, Q-10 | FR file mục 9.2; ACCEPTANCE_CRITERIA.md (ghi chú cuối file) |
| 9 | Sửa số liệu thống kê lỗi thời trong FR file (192 thành 199; Must 141, Should 45, Could 13; MVP 175, P2 21, hoãn 2, chưa chốt 1) | kiểm tra bằng lệnh | FR file mục 4 |

### 5.2 Số liệu sau cập nhật (199 FR, mỗi FR đúng một dòng)

| Verification Status | Số FR |
|---|---|
| AC_DEFINED | 109 |
| AC_ADDED_PHASE0 | 74 |
| AC_PROPOSED_PENDING_CLARIFICATION | 4 (FR-122, FR-181, FR-182, FR-183) |
| NEEDS_OWNER_CLARIFICATION (còn lại) | 12 (FR-006, 009, 038, 078, 088, 098, 153, 186, 200, 283, 284, 292) |
| **Tổng** | **199** |

Theo phần: Phần 1 = 51 / 19 / 0 / 6 (76 FR); Phần 2 = 34 / 22 / 4 / 2 (62 FR); Phần 3 = 24 / 33 / 0 / 4 (61 FR).
AC mới trong đợt này: AC-601..AC-614 (14) và AC-621..AC-623 (3) = 17 AC PROPOSED. Dải AC-615..AC-620 và AC-624..AC-699 dự trữ.

### 5.3 Câu hỏi còn mở (PENDING_OWNER_CLARIFICATION) phát sinh hoặc giữ lại từ OWR-2

- FR-122: OQ-122-1 (artifact bắt buộc), OQ-122-2 (export/download hay chỉ lưu và truy xuất), OQ-122-3 (phạm vi: Stage 3 hay mọi run).
- FR-181: OQ-181-1 (approval chờ lâu là incident hay operational alert), OQ-181-2 (quy tắc grouping; BA chỉ đề xuất ở AC-607).
- FR-182: OQ-182-1 (task cancelled), OQ-182-2 (trọng số bằng nhau), OQ-182-3 (định nghĩa baseline), OQ-182-4 (tiến độ theo giai đoạn so với theo task).
- FR-183: OQ-183-1 (nhiều metric hay một KPI), OQ-183-2 (công thức và trọng số nếu KPI).
- D-021: OQ-D021-1 (phạm vi kích hoạt OWN-AC-2), OQ-D021-2 (duyệt FR-293..298).
- FR-109 (công thức deterministic cụ thể), FR-113 (tên 9 UI node type và 11 engine kind), FR-164 (tên và phạm vi setting).
- Xung đột D-005 so với nhóm B (AC-022, GP-1); xung đột FR-076 "CLI-based" so với Q-17 (Agent SDK).
