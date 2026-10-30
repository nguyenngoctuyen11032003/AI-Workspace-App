# USER_JOURNEY — Hành trình người dùng AI WORKSPACE

> **Tác giả:** Agent 16 (UX Research & Product Designer) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
> **Tham chiếu:** `NAVIGATION_STRUCTURE.md` (route), `SCREEN_SPECIFICATION.md` (màn hình), `UX_RESEARCH_REPORT.md` (chiến lược tải nhận thức), `INTERACTION_DESIGN.md` (Agent 11).
> **Nhãn:** `[Đã xác minh]` · `[Đề xuất]` · `[Giả định]` · `[Cần người dùng phê duyệt]`.

---

## 1. Mục đích

Mô tả các hành trình end-to-end của **Director** (chủ sở hữu duy nhất, Phase 1) để: (a) xác định màn hình/route/sự kiện nào phải tồn tại ở mỗi bước, (b) lộ ra pain point và cơ hội thiết kế, (c) làm kịch bản cho usability test Phase 5/6 và acceptance test Phase 3–4. Mỗi hành trình có: giai đoạn, hành động, màn hình/route, điều người dùng nghĩ, pain point, cơ hội, evidence hệ thống cần hiển thị.

## 2. Phạm vi

- 1 persona chính: **Director** (chi tiết §3). Persona phụ **Viewer** chỉ được nhắc khi ảnh hưởng thiết kế.
- 8 hành trình: J1 Khởi động lần đầu đến sản phẩm đầu tiên (hành trình "xương sống"), J2 Quản lý 18 agent song song, J3 Agent bị kẹt / cần giúp, J4 Emergency stop, J5 Dạy agent một kỹ năng, J6 Vòng lặp revision sau khi reject, J7 Quay lại sau 8 giờ vắng mặt, J8 Giám sát từ điện thoại.
- Không bao gồm: hành trình cài đặt hạ tầng (Docker, CLI) ngoài phạm vi UI; hành trình nhiều người dùng.

## 3. Persona

**Director — "Tuyến"** `[Giả định]` từ brief: founder/tech lead một mình điều hành công ty phần mềm ảo; thành thạo kỹ thuật (đọc được diff, chạy test) nhưng không muốn ngồi trong IDE cả ngày; mục tiêu là ra sản phẩm với chất lượng kiểm soát được; nỗi sợ lớn nhất: agent "báo cáo xong" nhưng thực ra chưa, chi phí token vượt tầm, và mất kiểm soát khi 18 tiến trình chạy cùng lúc. Thiết bị: desktop 2 màn hình (chính), laptop, điện thoại để liếc trạng thái.

Nguyên tắc rút ra: **tin cậy > vui**, **nhìn thấy evidence > nhìn thấy animation**, **1 click tới nơi cần hành động**.

---

## 4. J1 — Từ lần mở đầu tiên đến sản phẩm được chấp nhận (hành trình xương sống)

| # | Giai đoạn | Hành động của Director | Màn hình / route | Người dùng nghĩ | Pain point | Cơ hội thiết kế | Evidence hệ thống phải hiển thị |
|---|---|---|---|---|---|---|---|
| 1 | Mở app lần đầu | Thấy Welcome | `/welcome` | "Mất bao lâu để thấy nó chạy thật?" | Wizard dài, hứa hẹn mơ hồ | 4 bước, ước lượng thời gian mỗi bước, "Skip" minh bạch | — |
| 2 | Connect Claude | Chọn Claude Code CLI; app detect; Test connection | `/welcome?step=1` → `/providers?connect=claude` | "Có nhận ra cài đặt của tôi không? Có lộ key không?" | Lỗi auth CLI khó hiểu; không biết model nào đang dùng | Detect version/path, hiển thị lệnh login để tự chạy (không chạy hộ), ping thật trả model + latency | Version CLI, model, latency ms, token của request test |
| 3 | Tạo agent đầu tiên | Chọn template "Business Analyst", đặt tên, test prompt | `/agents/new?step=1..6` | "Nó sẽ làm được gì? Tốn bao nhiêu?" | Form quá nhiều trường | Template điền sẵn, preview sống, chỉ 3 trường bắt buộc | Response test prompt thật; "Est. cost: —" khi chưa có dữ liệu |
| 4 | Tạo project từ prompt | Nhập mô tả sản phẩm, chọn template workflow Discovery→Release | `/projects/new` → `/projects/:p` | "Nó hiểu ý tôi chưa? Bước tiếp theo là gì?" | Prompt một lần rồi mất kiểm soát | Hiển thị workflow sẽ chạy + gate approval; task Discovery đầu tiên tạo thật và giao cho BA | Task #1 `in_progress`, agent BA `working` |
| 5 | Xem văn phòng | Mở Office, thấy BA ngồi gõ | `/office?focus=:ba` | "À, nó đang làm thật." | Không biết "gõ" nghĩa là gì | Ticker HUD "BA-01 reading repo README", panel hiển thị tool event thật | Tool event (read/write), thời lượng |
| 6 | Duyệt BA docs | Toast `approval.requested`; mở approval PRD | `/approvals/:id?tab=document` | "Tài liệu có đủ không? Thiếu gì?" | Đọc 20 trang trong UI chật; không hỏi lại được | Outline, open questions do agent liệt kê, tab Conversation hỏi agent trước khi quyết, inline comment | Version v1, checklist template BA, open questions |
| 7 | Duyệt design | Approval design spec + preview ảnh/wireframe | `/approvals/:id?tab=preview` | "Giống ý tôi chưa?" | Ảnh tĩnh khó so sánh với PRD | Side-by-side PRD section ↔ design; comment vào vùng ảnh `[Đề xuất]` | Link PRD section, version |
| 8 | Theo dõi Build | Dashboard progress; Office nhiều agent gõ | `/dashboard`, `/office` | "Tiến độ thật là bao nhiêu?" | % giả | `tasks 12/40`, `tests 30/30`, commit count; tooltip nguồn | Task/test/commit counts |
| 9 | Review code/test/security | Approval PR: diff, tests, findings | `/approvals/:id?tab=diff` → `tests` → `security` | "Có an toàn không? Test có thật không?" | Diff lớn; không biết test chạy khi nào | File tree, failing-first, finding → dòng code; checks pending hiển thị "đang chạy" thật | Run id, commit sha, pass/fail, findings |
| 10 | Preview | Mở sản phẩm chạy trong sandbox | `/projects/:p/preview` | "Dùng thử xem" | Preview là ảnh chụp | iframe app thật, console, device frames | Build id, commit, URL |
| 11 | Accept / Reject | Accept build hoặc Request changes kèm screenshot | `/projects/:p/preview` → approval | "Tôi muốn sửa 2 chỗ nhỏ" | Phải viết lại mô tả dài | Request changes tạo task revise với screenshot + URL + comment | Approval history |
| 12 | Revision | Agent sửa, nộp lại; Director duyệt v2 | `/approvals/:id?tab=diff` (v1→v2) | "Có đúng chỗ tôi yêu cầu không?" | Phải đọc lại toàn bộ | Diff chỉ phần thay đổi so với lần reject; map comment → thay đổi | Diff v1→v2, comment resolved |

**Thời gian mục tiêu `[Đề xuất]`:** bước 1–5 ≤ 15 phút trong usability test; bước 6 (duyệt PRD) ≤ 10 phút với tài liệu 10 trang.

**Điểm gãy nguy hiểm:** bước 2 (auth CLI), bước 6 (lần đầu thấy approval — nếu không hiểu Reject vs Request changes sẽ phá vòng lặp), bước 9 (diff quá lớn).

---

## 5. J2 — Quản lý 18 agent song song

**Bối cảnh:** 3 project, 18 agent, 22 task đang chạy, Director muốn một phiên kiểm tra 10 phút mỗi giờ.

| # | Giai đoạn | Hành động | Màn hình | Nghĩ | Pain point | Cơ hội | Evidence |
|---|---|---|---|---|---|---|---|
| 1 | Quét | Mở Dashboard, đọc "Needs you now" | `/dashboard` | "3 việc cần tôi, còn lại ổn" | 18 dòng feed nhảy liên tục | Needs-you gom và xếp hạng; feed throttle | Lý do mỗi item, thời gian chờ |
| 2 | Xử lý theo thứ tự | Click item 1 → approval; item 2 → resolve blocked | `/approvals/:id`, `/agents/:id` | "Xong cái này tới cái kia" | Mất ngữ cảnh khi nhảy màn | "Next needs-you" trong palette; sau khi resolve quay về Dashboard với item đã biến mất | — |
| 3 | Kiểm tra sức khỏe tổng | Agents list, sort Needs attention, lọc department | `/agents?view=list` | "Ai idle lâu? Ai chạy quá lâu?" | Không thấy outlier | Cột runtime có highlight > ngưỡng; idle > 15 phút gợi ý giao việc | Runtime, idle since |
| 4 | Cân bằng tải | Reassign task từ agent chậm sang idle | Dialog Reassign | "Agent nào phù hợp?" | Chọn nhầm agent thiếu skill | Gợi ý: idle + cùng dept + skill match + cost | Skill match, load |
| 5 | Kiểm soát chi phí | Liếc Cost card; mở Cost nếu bất thường | `/cost?groupBy=agent` | "Ai đang đốt token?" | Không biết tới cuối ngày | Ngân sách + hành động tự động khi vượt | Tokens, cost, budget % |
| 6 | Nhìn toàn cảnh | Mở Office fullscreen trên màn hình 2 | `/office` fullscreen | "Để đó, liếc là biết" | Animation gây xao nhãng | Reduced motion, chỉ bubble cho trạng thái cần người | State counts HUD |

**Cơ hội chính:** chế độ "Monitor" (Dashboard + Office fullscreen trên màn 2) và nhịp làm việc theo attention queue thay vì duyệt từng agent.

---

## 6. J3 — Agent bị kẹt / cần giúp

| # | Giai đoạn | Sự kiện/hành động | Màn hình | Nghĩ | Pain point | Cơ hội | Evidence |
|---|---|---|---|---|---|---|---|
| 1 | Phát hiện | WS `agent.blocked` → toast + bubble ⛔ trong Office + badge | mọi nơi | "Dev-03 kẹt gì?" | Toast biến mất, không biết đã lâu chưa | Toast persistent cho blocked, thời gian chờ chạy | Lý do kẹt dạng cấu trúc (`missing_env`, `permission`, `test_fail_loop`, `provider_error`, `ambiguous_spec`) `[Đề xuất]` |
| 2 | Hiểu | Mở Agent Panel / Agent Detail Overview "Needs you" | `/agents/:id?tab=overview` | "Nó cần gì chính xác?" | Phải đọc log dài | Card Needs-you có loại kẹt + câu hỏi agent + nút hành động tương ứng | Last 10 log + câu hỏi |
| 3 | Hành động | Cung cấp env (secret form), cấp permission 1 lần/luôn, trả lời câu hỏi, hoặc Reassign | cùng màn | "Chỉ cho lần này thôi" | Cấp quyền vĩnh viễn ngoài ý muốn | Lựa chọn rõ: Allow once / Allow for this task / Add to permissions | Ghi history permission |
| 4 | Xác nhận | Agent resume; bubble biến mất; task tiếp tục | Office / Agents | "Ổn rồi" | Không biết đã chạy lại chưa | State chuyển `working` + tool event mới trong ≤ 5s; nếu không → cảnh báo "chưa phản hồi" | State + event timestamp |
| 5 | Nếu lặp lại | Kẹt cùng lý do ≥ 2 lần | Agent Detail Performance | "Có vấn đề hệ thống" | Không thấy pattern | Gợi ý: sửa permission mặc định / thêm knowledge / teach skill (→ J5) | Số lần kẹt theo loại |

---

## 7. J4 — Emergency stop

| # | Giai đoạn | Hành động | Màn hình | Nghĩ | Pain point | Cơ hội | Evidence |
|---|---|---|---|---|---|---|---|
| 1 | Kích hoạt | `Ctrl+Shift+.` hoặc palette "Emergency stop all" hoặc nút đỏ Agents header | mọi nơi | "Dừng hết, ngay!" | Nút khó tìm lúc hoảng; hoặc quá dễ bấm nhầm | Luôn có trong palette + topbar menu; confirm gõ `STOP` | — |
| 2 | Xác nhận | Dialog liệt kê 14 agent đang chạy, task bị ảnh hưởng, lựa chọn: Stop all / Pause all (ưu tiên) / chỉ project X | `ConfirmDialog` | "Pause hay Stop?" | Không biết khác biệt | Giải thích 1 dòng: Pause giữ ngữ cảnh, Stop hủy tiến trình (không undo) | Danh sách tên |
| 3 | Thực thi | Hệ thống gửi stop song song; hiển thị tiến trình từng agent | Drawer kết quả | "Đã dừng thật chưa?" | Giả vờ dừng | Trạng thái per-agent `stopping → stopped` từ process exit thật; agent không phản hồi > 10s → "force kill" | Exit code, timestamp |
| 4 | Sau đó | Dashboard banner "Emergency stop lúc hh:mm — 14 agent dừng, 9 task dở" + nút Resume từng agent / Review changes | `/dashboard` | "Thiệt hại gì?" | Mất dấu việc dở | Danh sách task dở với diff chưa commit; nút "Stash changes" | Uncommitted changes per agent |

---

## 8. J5 — Dạy agent một kỹ năng

| # | Giai đoạn | Hành động | Màn hình | Nghĩ | Pain point | Cơ hội | Evidence |
|---|---|---|---|---|---|---|---|
| 1 | Nhận ra nhu cầu | Thấy Dev lặp lỗi review (không viết test) | Approval history / Performance | "Phải dạy nó" | Không biết bắt đầu từ đâu | Từ Performance "Review rejections" → nút "Teach a skill" | Số lần reject theo lý do |
| 2 | Tạo skill | Skill Editor: hướng dẫn, checklist, ví dụ | `/skills/new` | "Viết như viết cho nhân viên mới" | Không biết format | Template skill (When/Steps/Checklist/Examples), gợi ý từ reject comment | — |
| 3 | Thử | Test run với agent + sample input | Skill Editor › Test run | "Nó làm theo chưa?" | Không chạy thử được | Test run thật, hiển thị output + token | Output, latency |
| 4 | Gắn | Assign cho agent / department | `/skills` Assign | "Áp cho cả đội Eng" | Gắn nhầm | Dialog liệt kê agent, preview ảnh hưởng | — |
| 5 | Theo dõi | Performance của skill qua các task | `/skills` card metrics | "Có hiệu quả không?" | Không đo được | Success rate theo run, review rejection trước/sau | Runs, success % |

---

## 9. J6 — Vòng lặp revision sau khi reject

1. Director Reject hoặc Request changes với comment inline (`/approvals/:id`) → hệ thống tạo task `revise` gắn comment, workflow chuyển về node Revise `[Đã xác minh]` theo node Condition/Approval ở Workflow Builder.
2. Agent nhận task; Office hiển thị bubble ✋ biến mất, agent quay về bàn gõ; Dashboard "Needs you" bớt 1.
3. Agent nộp v2 → approval mới **liên kết** với approval cũ (History tab), diff chỉ v1→v2, mỗi comment cũ có trạng thái `addressed / not addressed` do agent đánh dấu + Director xác nhận `[Đề xuất]`.
4. Nếu Reject lần 3 cùng lý do → gợi ý Reassign hoặc Teach skill (J5).

**Pain point:** đọc lại từ đầu; **cơ hội:** diff theo comment.

---

## 10. J7 — Quay lại sau 8 giờ vắng mặt

1. Mở app → Dashboard: banner "Trong 8h qua: 31 task done, 2 blocked, 3 approvals chờ, $41.2, 1 provider sự cố (đã hồi phục)".
2. Activity Center filter "Needs action" → xử lý theo thứ tự.
3. Nếu có auto-pause do vượt ngân sách → hiển thị rõ và nút Resume với ngân sách mới.
**Cơ hội:** "Digest since last visit" tính từ `lastSeenAt` thật `[Đề xuất]`.

---

## 11. J8 — Giám sát từ điện thoại `[Cần người dùng phê duyệt]` (phạm vi mobile)

1. Nhận push/notification (Phase sau) → mở `/approvals/:id` mobile → đọc summary + checks → Approve/Request changes.
2. Tab Agents: list status-first, Pause/Resume; Stop có confirm.
3. Office read-only (zoom/pan), không edit.
**Pain point:** diff trên màn nhỏ → chỉ xem theo file + summary; cho phép "Approve on desktop later" (snooze).

---

## 12. Quyết định & lý do

| Quyết định | Lý do | Nhãn |
|---|---|---|
| Onboarding bắt buộc test connection thật trước khi tạo agent | Tránh "tạo xong không chạy" — điểm gãy #1. | `[Đề xuất]` |
| Approval có tab Conversation để hỏi agent trước khi quyết | Giảm reject vì hiểu nhầm; học từ pattern GitHub review conversation. | `[Đề xuất]` |
| Lý do kẹt có cấu trúc (enum) | Cho phép UI đưa đúng nút hành động; đo pattern lặp. | `[Đề xuất]` (cần Backend) |
| Emergency stop ưu tiên Pause all, Stop all gõ `STOP` | Giảm mất mát khi hoảng. | `[Đề xuất]` |
| Revision diff theo comment | Giảm thời gian duyệt v2 > 50%. | `[Đề xuất]` |

## 13. Dependencies
- Backend: enum `blocked_reason`, liên kết approval v1↔v2, `lastSeenAt`, permission grant scope (once/task/permanent), stop với exit code.
- Agent 11: ConfirmDialog typed-confirm pattern, Toast persistent, Banner.
- Workflow engine: node Revise/Condition on reject.

## 14. Rủi ro
- Nếu agent không trả lời câu hỏi "cần gì" có cấu trúc, J3 thoái hóa thành đọc log.
- Request changes tạo task tự động có thể sinh task rác nếu Director comment nhiều lần → gộp theo approval.
- Mobile: quyết định approve trên summary mà không đọc diff → rủi ro chất lượng; cần cảnh báo "bạn chưa mở diff".

## 15. Vấn đề chưa giải quyết
1. Director có muốn chat tự do với agent (như MVO) hay chỉ instruction gắn task? Đề xuất: instruction gắn task + Conversation trong approval; chat tự do là Phase sau.
2. Có cho phép Approve khi checks còn pending không? Đề xuất: cho phép với cảnh báo và ghi vào history.
3. Mobile Phase nào?

## 16. Acceptance Criteria
- [ ] J1 hoàn thành end-to-end trong môi trường thật với 1 provider trong usability test (≥ 4/5 người hoàn thành bước 1–6 không cần trợ giúp).
- [ ] Mỗi bước trong J1–J5 có route tồn tại và evidence liệt kê được hiển thị.
- [ ] J3: từ toast đến hành động resolve ≤ 3 click.
- [ ] J4: Stop all hiển thị trạng thái thật per-agent; không có agent "stopped" khi process còn sống.
- [ ] J6: approval v2 liên kết v1, diff v1→v2, comment có trạng thái addressed.
- [ ] J7: digest tính từ `lastSeenAt` thật.
