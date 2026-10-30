# TRACEABILITY — PART 1 (FR-001 … FR-100) — AI WORKSPACE

**Trạng thái:** DRAFT (Phase 0 finalization, chưa Owner phê duyệt) · **Ngày:** 2026-10-30
Nguồn: `FUNCTIONAL_REQUIREMENTS.md`, `ACCEPTANCE_CRITERIA.md` (AC-001…130), `AC_ADDENDUM_PART1.md` (AC-201…219).
Ghi chú: dải FR-001…100 chỉ có **76 FR** được định nghĩa (số còn lại để trống). Mỗi FR xuất hiện đúng một lần.
Verification Status: AC_DEFINED (AC có sẵn) · AC_ADDED_PHASE0 (AC mới ở addendum) · NEEDS_OWNER_CLARIFICATION; **`AC_PROPOSED_OWNER_DECIDED` (giá trị mới, OWR-3) = có AC PROPOSED (agent viết câu chữ), Owner đã xác nhận quyết định ở OWR-3 nên không còn câu hỏi chặn; AC vẫn là PROPOSED, không phải APPROVED, vì agent không được tự chuyển PROPOSED thành APPROVED; `WONT_MVP` (giá trị mới, OWR-3) = FR bị Owner loại khỏi MVP (WON'T DO IN MVP), giữ hàng truy vết và không viết AC.** `AC_PROPOSED_PENDING_CLARIFICATION` (OWR-2) nay không còn FR nào.

| FR ID | Priority | Acceptance Criteria ID(s) | Test Scenario | Expected Result | Verification Status |
|---|---|---|---|---|---|
| FR-001 | M | AC-201 | Tạo company có/không logo | Company lưu và hiển thị, tồn tại sau restart | AC_ADDED_PHASE0 |
| FR-002 | C | AC-202 | Chuyển giữa 2 workspace | Dữ liệu agent/skill/project cô lập theo workspace | AC_ADDED_PHASE0 |
| FR-003 | M | AC-203 | Tạo company mới | Đúng 12 phòng ban mặc định đúng tên | AC_ADDED_PHASE0 |
| FR-004 | M | AC-204 | CRUD phòng ban, xóa phòng còn agent | Xóa bị từ chối khi còn agent, thành công sau khi chuyển agent | AC_ADDED_PHASE0 |
| FR-005 | S | AC-205 | Chỉ định rồi đổi trưởng phòng | Mỗi phòng có đúng 1 trưởng phòng, hiển thị đúng | AC_ADDED_PHASE0 |
| FR-006 | S | AC-631, AC-632, AC-633 | Gán 2 quản lý cho 1 agent; chuỗi 50 cấp; gán tạo vòng lặp | Tối đa 1 quản lý trực tiếp; độ sâu không giới hạn; vòng lặp bị chặn, dữ liệu không đổi **[Nguồn: OWR-3 C; Owner xác nhận quyết định, AC PROPOSED]** | AC_PROPOSED_OWNER_DECIDED |
| FR-007 | S | AC-206 | Mở Org Chart | Cây 4 cấp đủ 12 phòng, mỗi agent đúng một lần | AC_ADDED_PHASE0 |
| FR-008 | S | AC-207 | Kéo-thả agent sang phòng/manager khác | Phòng và manager cập nhật, event được ghi | AC_ADDED_PHASE0 |
| FR-009 | S | AC-634, AC-635, AC-636 | Budget theo chi phí ước tính; chạm 80%; chạm 100% | Token theo dõi riêng; cảnh báo đúng 1 lần ở 80%; ở 100% dừng cấp task và gọi model mới, yêu cầu Owner quyết định **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |
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
| FR-038 | C | AC-637, AC-638, AC-639 | Meeting 3 agent; kết thúc meeting; Office không có event meeting | Meeting là thread thực lưu bền; PM ghi biên bản; animation chỉ theo event **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |
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
| FR-077 | M | AC-022 | Connect Codex đã login | Connected như Claude (UNVERIFIED nếu chưa cài Codex) **[OWR-3 A.8: hoãn khỏi MVP (D-005), MVP chỉ Claude; Owner-acknowledged. AC-022 không áp dụng cho MVP; hàng này giữ AC_DEFINED vì AC-022 vẫn tồn tại nguyên văn]** | AC_DEFINED |
| FR-078 | C | (không có, WON'T DO IN MVP) | Không có kịch bản kiểm thử trong MVP | Hoãn khỏi MVP theo D-006; giữ hàng truy vết, không viết AC **[Nguồn: OWR-3 C; OWR-2 D-006, Q-02]** | WONT_MVP |
| FR-079 | S | AC-219 | Adapter generic CLI với binary giả | Lệnh đúng tham số; output thành đúng event chuẩn | AC_ADDED_PHASE0 |
| FR-080 | M | AC-021, AC-022 | Connect bằng CLI đã login, không có API key | Kết nối thành công, không yêu cầu API key | AC_DEFINED |
| FR-081 | M | AC-021, AC-025, AC-034 | Gọi operation hỗ trợ và không hỗ trợ | Hoạt động đúng; không hỗ trợ trả NOT_SUPPORTED | AC_DEFINED |
| FR-082 | M | AC-021, AC-023 | Luồng phát hiện → login → hướng dẫn → kiểm tra lại | Hướng dẫn từng bước, kiểm tra lại hoạt động | AC_DEFINED |
| FR-083 | M | AC-023, AC-024 | CLI chưa cài / chưa login | Trạng thái Not installed / Not authenticated | AC_DEFINED |
| FR-084 | M | AC-024 | Kiểm tra UI khi chưa login | Không có ô nhập mật khẩu | AC_DEFINED |
| FR-085 | M | AC-031 | Rà soát mã adapter | Không có mã đọc file credential của CLI | AC_DEFINED |
| FR-086 | M | AC-029, AC-113 | Quét DB/log bằng scanner secret | 0 credential plaintext | AC_DEFINED |
| FR-087 | M | AC-030 | Bắt payload IPC/HTTP tới frontend | Không payload nào chứa credential | AC_DEFINED |
| FR-088 | S | AC-640, AC-641 | Quét tĩnh và runtime canary: không đọc/lưu credential (yêu cầu phủ định) | 0 mã đọc file credential, 0 ghi credential vào DB/file/OS credential manager, 0 sự kiện đọc canary từ tiến trình app **[Nguồn: OWR-3 C, D-008]** | AC_PROPOSED_OWNER_DECIDED |
| FR-089 | M | AC-027, AC-028 | Disconnect/Connect lại; phiên hết hạn | Kết nối lại được; SESSION_EXPIRED kèm hướng dẫn | AC_DEFINED |
| FR-090 | M | AC-029, AC-113 | Quét log sau 1 ngày chạy | Không credential trong log | AC_DEFINED |
| FR-091 | M | AC-032 | Kiểm tra sandbox hai agent cùng provider | Không credential; gọi provider qua adapter | AC_DEFINED |
| FR-092 | M | AC-033 | Mở Provider Hub | Hiển thị trạng thái, model, usage, lỗi, số session (dữ liệu thật) | AC_DEFINED |
| FR-093 | M | AC-003 | Mở dropdown model | Chỉ model thuộc ListModels | AC_DEFINED |
| FR-094 | M | AC-025 | SendMessage qua provider | StreamEvents phát event chuẩn realtime | AC_DEFINED |
| FR-095 | M | AC-026 | CancelTask khi session chạy | CLI dừng ≤ 5 s | AC_DEFINED |
| FR-096 | S | AC-012 | Mở Performance khi CLI không cấp usage | Usage ước lượng được đánh dấu "ước lượng" | AC_DEFINED |
| FR-097 | S | AC-034 | ResumeSession khi provider không hỗ trợ | NOT_SUPPORTED và fallback tạo session mới | AC_DEFINED |
| FR-098 | C | AC-642, AC-643 | Rà soát tĩnh không có đường API key; mất CLI auth (yêu cầu phủ định) | 0 trường/endpoint/biến môi trường API key; mất auth thì báo Not authenticated, không gọi thay thế **[Nguồn: OWR-3 C]** | AC_PROPOSED_OWNER_DECIDED |

## Tổng kết

| Verification Status | Số FR |
|---|---|
| AC_DEFINED | 51 |
| AC_ADDED_PHASE0 | 19 |
| AC_PROPOSED_OWNER_DECIDED (OWR-3: FR-006, 009, 038, 088, 098) | 5 |
| WONT_MVP (OWR-3: FR-078) | 1 |
| NEEDS_OWNER_CLARIFICATION | 0 |
| **Tổng** | **76** |

Số FR duy nhất trong dải FR-001…FR-100 = 76 (không phải 100; các số FR-012…020, 045…050, 069…075, 099, 100 chưa được định nghĩa trong FUNCTIONAL_REQUIREMENTS.md).

## Mã AC chưa định nghĩa đã xử lý (code → FR)

Không có mã AC nào trong dải AC-001…AC-065 được trích dẫn như tiêu chí cụ thể mà chưa định nghĩa. AC-016…020, 035…040, 065 chỉ xuất hiện như mô tả khoảng trống (CF-037, TESTING.md mục 7, ROADMAP rủi ro 5) và được giữ là số dự trữ, không bịa định nghĩa. Danh sách giải quyết: (rỗng).

## Câu hỏi cần Owner làm rõ (6) — ĐÃ ĐƯỢC TRẢ LỜI Ở OWR-3 mục C (2026-10-30)

Cập nhật OWR-3: Q-06 (FR-006), Q-09 (FR-009), Q-38 (FR-038), Q-88 (FR-088), Q-98 (FR-098) đã được Owner quyết định; AC mới AC-631..AC-643 (PROPOSED). Q-78 (FR-078): Owner xác nhận WON'T DO IN MVP (D-006), không viết AC, trạng thái WONT_MVP. Không còn câu hỏi chặn ở Part 1. Danh sách gốc giữ bên dưới để truy vết.

FR-006 (Q-06), FR-009 (Q-09), FR-038 (Q-38), FR-078 (Q-78), FR-088 (Q-88), FR-098 (Q-98). Chi tiết ở `AC_ADDENDUM_PART1.md` mục 4. Q-11 (loại event cho FR-008) là câu hỏi bổ sung không chặn AC.
