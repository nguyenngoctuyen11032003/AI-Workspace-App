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

**Cập nhật OWR-3 (2026-10-30):** sáu câu hỏi dưới đây đã được Owner trả lời ở OWR-3 mục C. AC-631..AC-643 (PROPOSED) trong `AC_ADDENDUM_PART4_OWR2.md` thay cho trạng thái thiếu thông tin; FR-078 là WON'T DO IN MVP (D-006), không viết AC. Bảng gốc giữ để truy vết.

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
