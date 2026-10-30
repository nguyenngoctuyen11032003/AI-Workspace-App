# ACCEPTANCE CRITERIA ADDENDUM — PART 4 (cập nhật OWR-2) — AI WORKSPACE

**Phiên bản:** 0.2 (cập nhật OWR-3) · **Vai trò:** BA · **Trạng thái:** DRAFT. Toàn bộ AC trong file này là **PROPOSED**, chưa có AC nào được Owner duyệt (không có AC nào APPROVED). Quyết định của Owner ở OWR-3 được ghi ở cột "Quyết định Owner xác nhận", nhưng câu chữ Given/When/Then do agent viết nên AC giữ PROPOSED · **Ngày:** 2026-10-30
**Nguồn:** `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md` (§1 Q-18 OWN-AC-1..3, §4 FR-122/181/182/183) và `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (A.3 đến A.6, A.12, mục C). Phase 0 chưa hoàn tất; Phase 1 chưa được Owner cho phép.

> Quy ước: Given/When/Then như `ACCEPTANCE_CRITERIA.md`. Dải mã dùng: AC-601..AC-614 (bốn FR dashboard/workflow), AC-615..AC-620 dự trữ, AC-621..AC-623 (D-021), AC-631..AC-657 (11 FR theo OWR-3 mục C). Nội dung OWR-2 §4 là **đề xuất của Owner, không phải trích nguyên văn FR**; AC chỉ diễn đạt lại đề xuất đó, không thêm phạm vi. Câu hỏi Owner chưa trả lời giữ nguyên là PENDING_OWNER_CLARIFICATION (OQ-xxx), AC không quyết định thay Owner.

## 1. FR-122, FR-181, FR-182, FR-183 (AC_PROPOSED_OWNER_DECIDED, cập nhật OWR-3)

**Cập nhật OWR-3 (2026-10-30, A.3 đến A.6):** Owner đã trả lời các câu hỏi mở. Mọi AC dưới đây vẫn là **PROPOSED**: quyết định của Owner được xác nhận ở OWR-3 nhưng câu chữ Given/When/Then do agent viết, và Owner đã yêu cầu agent không được chuyển AC PROPOSED thành APPROVED. Cột "Quyết định Owner xác nhận" cho biết mục OWR-3 làm căn cứ.

### 1.1 FR-122
Văn bản gốc trong repo: "Giai đoạn 3 System Design: Architect agent tạo thiết kế hệ thống." Quyết định Owner (OWR-3 A.3): artifact bắt buộc là output của các node `required` trong workflow definition; MVP lưu và xem trong workspace; export ZIP để phase sau. Vì quyết định nói theo "workflow definition", AC áp dụng cho mọi Workflow Run, trong đó Stage 3 là một trường hợp (xem ghi chú OQ-122-3 ở cuối mục).

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-601 | FR-122 | Workflow definition có node required R1, R2 và node không required O1; một Workflow Run đã hoàn thành cả ba node | Owner mở deliverable của run trong workspace | Deliverable package liệt kê đúng output của R1 và R2 (artifact bắt buộc), kèm output của O1 nếu có ở dạng không bắt buộc, run state và logs; truy xuất được theo workflow run id; xem được trực tiếp trong workspace (không cần rời ứng dụng); MVP không có chức năng export ZIP (không có nút, endpoint hay lệnh export artifact dạng ZIP) | Integration test + E2E; đối chiếu DB; kiểm tra tĩnh không có đường export ZIP | M | OWR-3 A.3 |
| AC-602 | FR-122 | Một run tạo ra artifact | Owner mở một artifact trong package | Artifact có metadata (tối thiểu: node/agent/task tạo ra, thời điểm tạo, gắn với run) và dữ liệu khớp execution evidence; không có artifact không có nguồn | Integration test | M | OWR-3 A.3 |
| AC-603 | FR-122 | Một run có node thất bại | Owner mở deliverable của run | Run được đánh dấu thất bại và chỉ ra node gây lỗi | Integration test với node lỗi cố ý | M | OWR-3 A.3 |
| AC-604 | FR-122 | Một run có ít nhất một node required chưa hoàn thành (chưa có output) | Run kết thúc luồng hoặc có yêu cầu đánh dấu hoàn thành | Run không được đánh dấu hoàn thành; chỉ hoàn thành khi mọi node required đã hoàn thành và có output; thiếu output của node required được báo rõ theo tên node | Integration test | M | OWR-3 A.3 |

Ghi chú còn lại (không chặn AC, không bịa quyết định): OQ-122-3 (đề xuất OWR-2 nói mọi Workflow Run, FR gốc chỉ nói Stage 3). OWR-3 A.3 dùng "workflow definition" nên AC viết cho mọi run; nếu Owner muốn giới hạn ở Stage 3, Owner cần nói rõ. OQ-122-1 và OQ-122-2 đã được trả lời bởi OWR-3 A.3.

### 1.2 FR-181
Văn bản gốc: "Overview dashboard: số agent theo trạng thái, project đang chạy, task theo status, approval chờ, chi phí hôm nay/tháng, sự cố." Quyết định Owner (OWR-3 A.4): approval chờ lâu là operational alert, không phải incident; incident cùng loại và cùng đối tượng trong cửa sổ 15 phút được grouping có bộ đếm.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-605 | FR-181 | Có sự kiện vận hành bất thường thuộc các loại: execution error, timeout, provider rate limit, sandbox failure, lost connection/heartbeat | Sự kiện xảy ra và có bằng chứng trong event store | Dashboard hiển thị incident có đủ ID, type, severity, timestamp, target (task/agent/workflow/infrastructure bị ảnh hưởng) và status. Approval chờ lâu KHÔNG nằm trong danh sách loại incident (xem AC-651) | Integration test mỗi loại + E2E | M | OWR-3 A.4 |
| AC-606 | FR-181 | Không có bằng chứng sự cố trong event store (workspace mới hoặc hệ thống bình thường) | Owner mở dashboard | Không có incident nào được tạo hoặc hiển thị; không suy diễn incident khi thiếu bằng chứng; hiển thị 0 hoặc empty state | Integration test; liên kết AC-316 | M | OWR-3 A.4 (giữ từ đề xuất OWR-2 §4) |
| AC-607 | FR-181 | Ba incident cùng `type` và cùng `target` xảy ra ở phút 0, 5 và 12; một incident thứ tư cùng type và target xảy ra ở phút 30 (quá cửa sổ 15 phút tính từ `first_seen` của nhóm); một incident khác type hoặc khác target xảy ra ở phút 6 | Hệ thống ghi nhận các incident | Ba incident đầu gom thành một incident có `occurrence_count` = 3, `first_seen` = phút 0, `last_seen` = phút 12; incident ở phút 30 tạo incident mới có `occurrence_count` = 1; incident khác type/target không bị gom (hai incident riêng); dashboard hiển thị bộ đếm. Cửa sổ = 15 phút (cấu hình hằng số được ghi trong tài liệu). Cách đo cửa sổ (từ `first_seen` của nhóm) là diễn giải của BA, QA/Owner có thể điều chỉnh | Unit test hàm grouping (ranh giới 14:59 và 15:01) + integration | M | OWR-3 A.4 |

### 1.3 FR-182
Văn bản gốc: "Project dashboard: tiến độ theo giai đoạn, task, bug, chi phí, thời gian." Quyết định Owner (OWR-3 A.5): task `cancelled` loại khỏi cả tử số và mẫu số; baseline gồm task ban đầu và các thay đổi phạm vi đã được duyệt; chấp thuận trọng số bằng nhau cho MVP.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-608 | FR-182 | Project có 10 task ban đầu; 2 task thêm qua thay đổi phạm vi đã được duyệt; 1 task thêm không qua duyệt phạm vi | Hệ thống tính tiến độ | Baseline = 12 task (10 ban đầu + 2 thay đổi phạm vi đã duyệt); task thêm không qua duyệt không nằm trong baseline; Progress = 100 × (tổng trọng số task `completed` / tổng trọng số task trong baseline), khớp phép tính độc lập từ DB | Unit test công thức + integration | M | OWR-3 A.5 |
| AC-609 | FR-182 | Baseline có task `completed`, `running`, `blocked` | Hệ thống tính tiến độ | Chỉ task `completed` được tính vào tử số; `running` và `blocked` không được tính vào tử số và vẫn nằm trong mẫu số | Unit test | M | OWR-3 A.5 (giữ từ OWR-2 §4) |
| AC-610 | FR-182 | Baseline 10 task: 4 `completed`, 2 `cancelled`, 4 còn lại `running` hoặc `blocked` | Hệ thống tính tiến độ | Hai task `cancelled` bị loại khỏi cả tử số và mẫu số: Progress = 100 × 4/8 = 50%; tooltip hoặc chú thích ghi số task `cancelled` bị loại | Unit test với bộ dữ liệu cố định | M | OWR-3 A.5 |
| AC-611 | FR-182 | MVP chưa có hệ thống estimate đáng tin cậy; hai task có độ dài khác nhau | Hệ thống tính tiến độ | Mọi task trong baseline có trọng số bằng nhau (1); hoàn thành một task dài hay ngắn đều làm Progress tăng cùng một lượng = 100/(số task baseline chưa bị cancelled) | Unit test | M | OWR-3 A.5 |

Ghi chú còn lại (không chặn AC): OQ-182-4 (FR gốc nói "tiến độ theo giai đoạn", đề xuất tính theo task). OWR-3 không trả lời trực tiếp cách quy đổi tiến độ giai đoạn; AC chỉ kiểm tiến độ theo task ở cấp project. Không bịa công thức theo giai đoạn.

### 1.4 FR-183
Văn bản gốc: "Agent dashboard: hiệu suất, token, cost, lỗi, thời gian làm việc." Quyết định Owner (OWR-3 A.6): dashboard nhiều metric, không tạo KPI tổng hợp trong MVP.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-612 | FR-183 | Có dữ liệu thực thi thật | Owner mở dashboard hiệu suất | Hiển thị các metric riêng biệt: lead time, execution time, success rate, retry rate, throughput; mỗi metric có giá trị và nhãn riêng; KHÔNG có điểm số hay KPI tổng hợp duy nhất nào gộp các metric (kiểm tra UI và API không có trường điểm tổng hợp); các metric tách biệt với Progress (không dùng Progress làm Performance) | E2E + đối chiếu truy vấn DB độc lập; kiểm tra schema API | M | OWR-3 A.6 |
| AC-613 | FR-183 | Có dữ liệu theo nhiều agent, task, workflow và thời gian | Owner chọn breakdown | Chỉ số được tách theo Agent, Task, Workflow và theo thời gian; tổng các nhóm khớp tổng chung | Integration test | M | OWR-3 A.6 |
| AC-614 | FR-183 | Provider có hoặc không cung cấp usage | Owner mở dashboard | Token và cost chỉ hiển thị khi provider cung cấp dữ liệu; nếu không có thì hiển thị trạng thái không có dữ liệu (không tự bịa). Giá trị ước lượng, nếu có theo FR-096, phải được đánh dấu "ước lượng" | Integration test | M | OWR-3 A.6 (giữ từ OWR-2 §4) |

Ghi chú: OQ-183-1 đã được trả lời (nhiều metric); OQ-183-2 không còn áp dụng (không có KPI tổng hợp trong MVP). Success rate ở AC-612 dùng định nghĩa của FR-186 (AC-646).

## 1A. 11 FR bổ sung theo OWR-3 mục C (AC-631 đến AC-657) và FR-078

Tất cả AC dưới đây là **PROPOSED** (Owner xác nhận quyết định ở OWR-3 mục C, agent viết câu chữ Given/When/Then). Trạng thái truy vết: AC_PROPOSED_OWNER_DECIDED. Dải AC-631..AC-657 liên tục, mỗi mã định nghĩa đúng một lần.

### 1A.1 FR-006 Quản lý trực tiếp (OWR-3 C)
Quyết định: một quản lý trực tiếp mỗi nhân sự, không giới hạn độ sâu, chặn vòng lặp. "Nhân sự" được hiểu là agent trong company (BA diễn giải, FR gốc dùng "reporting chain").

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-631 | FR-006 | Agent A đã có quản lý trực tiếp M1 | Owner gán thêm M2 làm quản lý của A | Hệ thống chỉ giữ đúng một quản lý trực tiếp cho A: thao tác thay M1 bằng M2 (sau xác nhận) hoặc bị từ chối với thông báo rõ; không bao giờ có A với 2 quản lý; ràng buộc nằm ở tầng dữ liệu (truy vấn đếm quản lý của mọi agent luôn ≤ 1) | Integration test + ràng buộc DB | S | OWR-3 C (FR-006) |
| AC-632 | FR-006 | Chuỗi quản lý gồm 50 agent nối tiếp (agent n báo cáo agent n-1) | Owner mở reporting chain của agent cấp thấp nhất | Chuỗi được tạo thành công, không bị từ chối vì độ sâu; reporting chain truy vấn ra đủ 50 cấp theo đúng thứ tự; không có hằng số giới hạn độ sâu trong mã (kiểm tra tĩnh) | Integration test + rà soát mã | S | OWR-3 C (FR-006) |
| AC-633 | FR-006 | A báo cáo cho B, B báo cáo cho C | Owner (hoặc agent) gán A làm quản lý của C; và gán B làm quản lý của B | Cả hai thao tác bị chặn với lỗi nêu rõ vòng lặp; dữ liệu quản lý không thay đổi (0 khác biệt trước/sau); có audit ghi lần thử bị chặn | Unit test thuật toán phát hiện vòng + integration | S | OWR-3 C (FR-006) |

### 1A.2 FR-009 Ngân sách phòng ban (OWR-3 C)
Quyết định: budget tính theo chi phí ước tính quy đổi từ token và các khoản phí liên quan; theo dõi token riêng; cảnh báo 80%; vượt 100% dừng cấp task và gọi model mới, yêu cầu Owner quyết định. Danh mục "các khoản phí liên quan" và các lựa chọn cụ thể của quyết định Owner không được OWR-3 liệt kê, AC chỉ kiểm theo bảng giá/cấu hình phí hiện có và việc yêu cầu quyết định, không bịa danh mục.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-634 | FR-009 | Phòng ban D có budget 100 đơn vị tiền; usage gồm 1.000.000 token và các khoản phí liên quan theo bảng giá cấu hình | Owner xem budget của D | Mức đã dùng = chi phí ước tính quy đổi từ token và các khoản phí liên quan, khớp phép tính độc lập; token được theo dõi và hiển thị riêng thành một số liệu khác (không dùng token làm đơn vị budget); chi phí ước lượng được đánh dấu "ước lượng" (FR-096) | Unit test quy đổi + integration | S | OWR-3 C (FR-009) |
| AC-635 | FR-009 | Budget 100, đã dùng 79 | Usage tăng lên 80 (chạm 80%) | Hệ thống phát đúng một cảnh báo ngưỡng 80% cho phòng ban D (không lặp mỗi lần usage tăng thêm), ghi audit/event; chưa dừng cấp task và gọi model | Integration test (ranh giới 79 → 80, tăng tiếp 81 không cảnh báo lại) | S | OWR-3 C (FR-009) |
| AC-636 | FR-009 | Budget 100, đã dùng 99, task READY và agent thuộc D đang chờ gọi model | Usage chạm hoặc vượt 100% | Hệ thống dừng cấp task mới và dừng phát lời gọi model mới cho phòng ban D; hiển thị yêu cầu Owner quyết định; không tự động tiếp tục khi chưa có quyết định của Owner (task READY vẫn không được cấp); quyết định của Owner được ghi audit. Lựa chọn cụ thể (ví dụ tăng budget) không do OWR-3 quy định | Integration test với FakeProvider; kiểm tra không có lời gọi model mới sau mốc 100% | S | OWR-3 C (FR-009) |

### 1A.3 FR-038 Meeting (OWR-3 C)
Quyết định: meeting là thread thực, PM ghi biên bản; animation chỉ phản ánh sự kiện. Tên event theo danh mục FR-117 (MEETING_STARTED, MEETING_ENDED) và chuẩn đặt tên event do tài liệu event quy định (OWR-3 A.16).

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-637 | FR-038 | Một agent tạo meeting chủ đề T với 3 agent tham gia | Các agent trao đổi qua meeting | Meeting là một thread thực được lưu bền (meeting id, chủ đề, danh sách tham gia, các message theo thứ tự, thời điểm bắt đầu/kết thúc); sau restart app thread và message còn nguyên; event meeting bắt đầu/kết thúc được ghi | Integration test + chaos test (kill process) | C | OWR-3 C (FR-038) |
| AC-638 | FR-038 | Meeting kết thúc | Hệ thống hoàn tất meeting | PM agent ghi biên bản và biên bản lưu liên kết với meeting id (người ghi = PM agent, có thời điểm); meeting không có biên bản của PM thì không được đánh dấu đã đóng hoàn chỉnh hoặc hiển thị trạng thái "thiếu biên bản" | Integration test | C | OWR-3 C (FR-038) |
| AC-639 | FR-038 | Không có thread meeting hoặc event meeting nào trong event store | Owner mở Virtual Office trong 5 phút | Không có animation meeting hoặc nhân vật vào phòng Meeting; khi event meeting bắt đầu được phát thì animation meeting xuất hiện và mỗi chuyển động đối chiếu được với event id; animation không tự tạo ra thread hay biên bản | Test replay xác định (liên kết AC-089..AC-091) | C | OWR-3 C (FR-038) |

### 1A.4 FR-088 Credential: yêu cầu phủ định (OWR-3 C, D-008)
Quyết định: ứng dụng không lưu hoặc tự ý đọc credential theo D-008. Văn bản FR cũ ("nếu phải lưu credential thì dùng OS credential manager") không còn là hành vi MVP. Đây là **yêu cầu phủ định**: xác minh bằng test chứng minh không có hành vi, không phải bằng demo tính năng.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-640 | FR-088 | Mã nguồn ứng dụng (backend, adapter, shell, frontend) và danh sách đường dẫn credential của CLI (ví dụ thư mục cấu hình đăng nhập của Claude CLI; danh sách chốt trong test) | Chạy quét tĩnh/rà soát mã và quét DB, file cấu hình, log, tệp tạm sau một phiên kết nối và chạy task | 0 chỗ trong mã mở, đọc hoặc sao chép file credential của CLI; 0 lời gọi ghi vào OS credential manager (Windows Credential Manager) hoặc keyring; 0 cột/bảng/trường lưu credential; 0 credential (quét regex secret) trong DB, file, log | Static scan (grep/AST) trong CI + secret scanner; liên kết AC-029, AC-031, AC-113 | S | OWR-3 C (FR-088); D-008 | **[OWR-3 C (FR-088, FR-098): ứng dụng không lưu hoặc tự đọc credential provider (D-008); secret khác phải làm rõ ở Q-21]**
| AC-641 | FR-088 | Môi trường test có thư mục credential giả (canary) ở đường dẫn credential của CLI, với giám sát truy cập file | Thực hiện luồng kết nối provider, chat, chạy task và logout | Tiến trình ứng dụng không mở/đọc canary (log giám sát = 0 sự kiện đọc đường dẫn đó từ tiến trình app, ngoại trừ chính CLI/SDK do provider sở hữu chạy dưới quyền của chính nó); ứng dụng không hiển thị hay nhận credential trong payload (liên kết AC-030) | Runtime test với canary + file-access monitor | S | OWR-3 C (FR-088); D-008 |

### 1A.5 FR-098 Không có API key fallback: yêu cầu phủ định (OWR-3 C)
Quyết định: không có API key fallback trong MVP. Yêu cầu phủ định, xác minh bằng test.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-642 | FR-098 | Toàn bộ UI, API, schema cấu hình, biến môi trường và mã adapter của bản MVP | Rà soát tĩnh và duyệt UI Provider Hub, Settings, Agent Builder | 0 trường nhập API key; 0 endpoint/lệnh/biến môi trường nhận API key của provider; 0 đường mã dùng API key để gọi provider; 0 mục cấu hình "fallback" cho API key | Static scan (grep tên tham số, biến môi trường, route) + rà soát UI bằng E2E | C | OWR-3 C (FR-098) |
| AC-643 | FR-098 | CLI auth của provider không khả dụng (chưa login hoặc hết hạn) | Owner thử kết nối và chạy task | Trạng thái hiển thị là "Not authenticated"/SESSION_EXPIRED kèm hướng dẫn login (FR-082, AC-024, AC-028); không có lời gọi provider nào được thực hiện bằng cách khác; không có gợi ý nhập API key; task không chạy cho tới khi CLI auth hoạt động trở lại | Integration test với FakeProvider mô phỏng mất auth + chặn mạng để xác nhận 0 lời gọi thay thế | C | OWR-3 C (FR-098) |

### 1A.6 FR-153 Một tầng trong MVP (OWR-3 C)
Quyết định: một tầng trong MVP; không tự mở rộng.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-644 | FR-153 | Bản MVP với 14 phòng của FR-151 | Owner mở Virtual Office | Đúng 1 tầng chứa cả 14 phòng; không có điều khiển chuyển tầng, không có khái niệm tầng thứ hai trong UI và schema layout của MVP | E2E + kiểm tra schema | C | OWR-3 C (FR-153) |
| AC-645 | FR-153 | Workspace có 18 agent và có thêm phòng ban tùy chỉnh | Owner thêm phòng ban, agent hoặc đổi layout | Hệ thống không tự tạo tầng mới hay mở rộng số tầng; mọi phòng vẫn nằm trên tầng duy nhất; nếu không đủ chỗ thì hiển thị cảnh báo thay vì tạo tầng | E2E + integration test | C | OWR-3 C (FR-153) |

### 1A.7 FR-186 Quality dashboard: success rate (OWR-3 C)
Quyết định: success rate = completed / (completed + failed) × 100%; không tính running, blocked, cancelled; mẫu số bằng 0 thì hiển thị N/A.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-646 | FR-186 | Tập task có 6 `completed` và 2 `failed` | Owner mở Quality dashboard | Success rate = 6/(6+2) × 100% = 75%; khớp phép tính độc lập từ DB; công thức nêu trong tooltip | Unit test công thức + integration | S | OWR-3 C (FR-186) |
| AC-647 | FR-186 | Tập task có 6 `completed`, 2 `failed`, 3 `running`, 2 `blocked`, 4 `cancelled` | Hệ thống tính success rate | Kết quả vẫn là 75%: `running`, `blocked`, `cancelled` không nằm trong tử số lẫn mẫu số; đổi trạng thái một task `running` sang `completed` thì kết quả đổi theo công thức | Unit test với bộ dữ liệu cố định | S | OWR-3 C (FR-186) |
| AC-648 | FR-186 | Tập task không có task `completed` và không có task `failed` (chỉ `running`, `blocked`, `cancelled`, hoặc rỗng) | Owner mở Quality dashboard | Success rate hiển thị "N/A" (không phải 0% và không lỗi chia cho 0) | Unit test + E2E | S | OWR-3 C (FR-186) |

Ghi chú còn lại (không chặn AC): FR-186 còn liệt kê test pass rate, security findings, reject rate. OWR-3 chỉ định nghĩa success rate. OQ-186-1: công thức và mẫu số của test pass rate và reject rate (và cách đếm security findings) chưa được Owner định nghĩa; AC-646..648 chỉ phủ success rate.

### 1A.8 FR-200 Nhắc approval (OWR-3 C, A.12)
Quyết định: N = 24 giờ, là operational alert. Tách `remind_at` (nhắc Owner sau 24 giờ) và `expires_at` (15 phút với approval của hành động quan trọng).

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-649 | FR-200 | Approval thường (không phải hành động quan trọng) được tạo lúc t0 và vẫn WAITING | Đồng hồ test đến t0 + 24 giờ - 1 phút, rồi t0 + 24 giờ | `remind_at` = t0 + 24 giờ lưu trong bản ghi approval; trước mốc: không có nhắc; tại mốc: xuất hiện đúng một operational alert nhắc Owner (hiển thị trong ứng dụng, nêu approval nào và thời gian chờ); approval vẫn WAITING, không tự approve/reject/hủy | Integration test với đồng hồ giả (clock injection) | C | OWR-3 C (FR-200), A.12 |
| AC-650 | FR-200 | Approval có alert nhắc ở AC-649 đã phát | Owner mở dashboard sự cố (FR-181) và danh sách incident | Alert nhắc approval không xuất hiện như incident và không tăng bộ đếm incident; nằm ở mục operational alert riêng; Owner approve hoặc reject thì alert được xóa | Integration test + E2E; liên kết AC-605 | C | OWR-3 C (FR-200), A.4 |
| AC-651 | FR-200 | Approval của hành động quan trọng được tạo lúc t0 | Xem bản ghi approval và đồng hồ test đến t0 + 15 phút | Bản ghi có `expires_at` = t0 + 15 phút, là trường riêng biệt với `remind_at` (không dùng chung một trường); đến `expires_at` mà chưa được duyệt thì approval không được coi là đã duyệt và hành động không được thực thi (BA diễn giải: hết hạn không ngầm đồng nghĩa với duyệt; OWR-3 không nêu hành vi chi tiết sau khi hết hạn) | Integration test với đồng hồ giả | C | OWR-3 A.12 |

### 1A.9 FR-283 Usability (OWR-3 C)
Quyết định: dùng checklist usability 10 tiêu chí (`docs/design/USABILITY_AUDIT_CHECKLIST.md` mục 1: 10 bước của chủ sở hữu); không tự xóa P0 nếu chưa có bằng chứng QA. Mức P0 xuất hiện ở các mục M và O của checklist (mục 2, 3), mục 1 không gán mức.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-652 | FR-283 | Một màn hình trong `SCREEN_SPECIFICATION.md` đã triển khai | QA kiểm usability màn hình đó | Có một bản checklist riêng cho màn hình, đủ 10 tiêu chí (bố cục, typography, spacing, màu sắc, responsive, accessibility, trạng thái, tương tác thật, nhất quán, người mới), mỗi tiêu chí ghi Pass, Fail hoặc N/A kèm bằng chứng (ảnh chụp, test hoặc kết quả công cụ); tiêu chí thiếu kết quả hoặc thiếu bằng chứng thì màn hình chưa được tính là đã kiểm | Rà soát tài liệu QA + so khớp với checklist | S | OWR-3 C (FR-283) |
| AC-653 | FR-283 | Checklist của một màn hình có mục mức P0 ở trạng thái Fail (hoặc chưa kiểm) | Ai đó (kể cả agent) đánh dấu mục P0 là đã xử lý, xóa mục, hạ mức hoặc đánh dấu màn hình hoàn thành | Thao tác bị chặn hoặc bị từ chối ở quy trình QA: mục P0 chỉ được chuyển sang Pass khi có bằng chứng QA đính kèm (đường dẫn ảnh, test hoặc báo cáo) và người QA ghi nhận; không có bằng chứng thì màn hình giữ trạng thái chưa hoàn thành; lịch sử thay đổi mục P0 truy vết được | Rà soát quy trình QA trên một màn hình mẫu (kiểm thử quy trình, thử xóa P0 không bằng chứng) | S | OWR-3 C (FR-283) |

### 1A.10 FR-284 Ngôn ngữ (OWR-3 C, B X-1)
Quyết định: tiếng Việt mặc định, hỗ trợ tiếng Anh. Định dạng ngày/số theo locale và phạm vi chuỗi (UI, lỗi, log agent) không được OWR-3 nêu; AC chỉ phủ chuỗi giao diện.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-654 | FR-284 | Cài mới, chưa đặt ngôn ngữ | Owner mở ứng dụng lần đầu | Giao diện hiển thị tiếng Việt; locale mặc định lưu là `vi`; 0 chuỗi giao diện hiển thị khóa i18n thô | E2E Playwright + quét chuỗi | C | OWR-3 C (FR-284); B X-1 |
| AC-655 | FR-284 | Ứng dụng đang ở tiếng Việt | Owner đổi ngôn ngữ sang tiếng Anh trong Settings | Mọi chuỗi giao diện của các màn hình MVP chuyển sang tiếng Anh ngay hoặc sau khi tải lại; lựa chọn được lưu qua restart; tập khóa tiếng Việt và tiếng Anh trùng nhau (CI kiểm 0 khóa thiếu ở mỗi ngôn ngữ); chuyển lại tiếng Việt hoạt động | E2E + kiểm tra CI catalog i18n | C | OWR-3 C (FR-284) |

### 1A.11 FR-292 Sao lưu: chỉ hướng dẫn PostgreSQL dump (OWR-3 C)
Quyết định: MVP chỉ hướng dẫn PostgreSQL dump; workspace export để phase sau.

| ID | FR | Given | When | Then | Verification | Pri | Quyết định Owner xác nhận |
|---|---|---|---|---|---|---|---|
| AC-656 | FR-292 | Bản MVP và tài liệu vận hành | Rà soát tài liệu và ứng dụng | Có tài liệu hướng dẫn tiếng Việt từng bước tạo PostgreSQL dump (lệnh, tham số, nơi lưu, lưu ý không chứa credential) và từng bước khôi phục; ứng dụng MVP không có chức năng export workspace (không có nút, endpoint hay lệnh export workspace); chức năng này được ghi là phase sau | Rà soát tài liệu + kiểm tra tĩnh UI/API | C | OWR-3 C (FR-292) |
| AC-657 | FR-292 | Môi trường test có dữ liệu mẫu thật trong PostgreSQL (agent, task, event) | Một người thực hiện đúng từng bước của tài liệu để tạo dump rồi khôi phục vào PostgreSQL trống | Dump tạo được; sau khôi phục số dòng của các bảng chính bằng số dòng nguồn (0 khác biệt) và ứng dụng khởi động được trên bản khôi phục; dump không chứa credential (quét regex = 0) | Thử nghiệm thủ tục trên môi trường test (chưa chạy trong Phase 0) | C | OWR-3 C (FR-292) |

### 1A.12 FR-078: WONT_MVP (không có AC)
FR-078 (Adapter OpenHands): OWR-3 C xác nhận WON'T DO IN MVP theo D-006. Không viết AC. Hàng truy vết được giữ với trạng thái `WONT_MVP` (định nghĩa ở chú giải file truy vết). Nếu Owner đưa FR trở lại phạm vi thì mới viết AC.

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
