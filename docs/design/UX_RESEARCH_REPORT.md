# UX_RESEARCH_REPORT — Nghiên cứu UX cho AI WORKSPACE

> **Tác giả:** Agent 16 (UX Research & Product Designer) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
> **Tham chiếu:** `NAVIGATION_STRUCTURE.md`, `SCREEN_SPECIFICATION.md`, `USER_JOURNEY.md`, `USABILITY_AUDIT_CHECKLIST.md`; Design System của Agent 11 (`DESIGN_SYSTEM.md`, `UI_UX_GUIDELINES.md`); `docs/research/MY_VIRTUAL_OFFICE_ANALYSIS.md` (Research Agent 03).
> **Nhãn:** `[Đã xác minh]` = đọc trực tiếp từ README/screenshot repo tham chiếu hoặc brief · `[Đề xuất]` · `[Giả định]` = dựa trên hiểu biết chung về sản phẩm tham chiếu (Linear, Vercel…) không được xác minh lại trong phiên này · `[Cần người dùng phê duyệt]`.

---

## 1. Mục đích

Cung cấp cơ sở nghiên cứu cho các quyết định UX của AI WORKSPACE: (1) phân tích heuristic các sản phẩm tham chiếu để rút **pattern** (không sao chép), (2) phân tích UX hai sản phẩm văn phòng ảo (Pixel Agents, my-virtual-office) từ README và screenshot, (3) chiến lược giảm tải nhận thức khi điều hành 18 agent, (4) rủi ro usability của Virtual Office, (5) khuyến nghị, (6) kế hoạch usability test cho Phase 5/6.

## 2. Phạm vi & phương pháp

- **Phương pháp:** heuristic review (Nielsen 10 heuristics + cognitive load theory), desk research trên README/screenshot có sẵn trong workspace, phân tích nhiệm vụ (task analysis) từ brief.
- **Bằng chứng trực tiếp đã đọc `[Đã xác minh]`:** `D:\Code\Tuyen\Agent management\pixel-agents\README.md`; `D:\Code\Tuyen\Agent management\my-virtual-office\README.md`; `my-virtual-office\screenshot.png` (xem được bằng Read tool); `docs/research/MY_VIRTUAL_OFFICE_ANALYSIS.md`.
- **Không có:** phỏng vấn người dùng, dữ liệu analytics (chưa có sản phẩm). Các nhận định về Linear/Vercel/Raycast/Notion/Stripe/GitHub/Figma dựa trên hiểu biết chung về pattern công khai của các sản phẩm này `[Giả định]`, mô tả ở mức pattern để tránh sao chép.
- **Ngoài phạm vi:** visual design (Agent 11), benchmark hiệu năng (Agent Performance).

## 3. Phân tích heuristic các sản phẩm tham chiếu (pattern, không sao chép)

| Sản phẩm | Pattern đáng học | Áp dụng vào AI WORKSPACE | Không áp dụng / lưu ý |
|---|---|---|---|
| **Linear** | Keyboard-first (chord `G` + chữ, `C` tạo); command palette là trung tâm; list density cao, nhiều view (board/list) chung một filter model; trạng thái bằng icon + màu nhất quán; "triage inbox" tách việc cần xử lý khỏi việc đang theo dõi | Shortcut §9 `NAVIGATION_STRUCTURE.md`; Task Center 4 view chung filter; Approvals/Needs-you như inbox triage; icon trạng thái agent/node | Mức tối giản thị giác của Linear có thể làm agent-state khó phân biệt khi 18 dòng — ta thêm StatusDot + label text |
| **Vercel** | Deployment timeline với trạng thái rõ (Queued/Building/Ready/Error), log stream theo bước, preview URL per deployment, "evidence" (commit sha, build time) ngay trên card | Product Preview và QA runs: mỗi run gắn commit sha, thời lượng, log; Workflow node states | Vercel là single-purpose; dashboard của ta phải gom nhiều loại đối tượng → cần phân cấp mạnh hơn |
| **Raycast** | Một input đa năng với tiền tố/alias; kết quả nhóm theo section; hành động phụ qua `Ctrl+K` trong kết quả; feedback tức thì | Command Palette tiền tố `@ # ! /`; action submenu trên kết quả agent/task | Raycast là launcher toàn OS; ta giữ phạm vi trong app |
| **Notion** | Progressive disclosure (toggle, nested); property-based filtering; empty-state có template; inline edit | Agent Detail/Approval document tab (outline + collapse); empty-state với template team/workflow; inline-edit tên | Tự do bố cục của Notion gây mất nhất quán — ta cố định layout màn chuyên nghiệp |
| **Stripe** | Dashboard số liệu với định nghĩa rõ (tooltip công thức), so sánh kỳ trước chỉ khi có dữ liệu, bảng chi tiết dưới chart, test/live mode phân biệt rõ ràng | MetricCard có tooltip công thức & nguồn; Cost Management có bảng giá + ngày; Sandbox/permission mode hiển thị tường minh | Stripe nhiều cấp menu — ta giới hạn 14 group, 2 cấp |
| **GitHub** | PR review: diff file tree, comment theo dòng, "Files changed" vs "Conversation" tách tab, checks status (pending/pass/fail) có evidence, "Request changes/Approve/Comment" rõ nghĩa | Approval Center tab Diff/Conversation/Tests/Security; 3 hành động cùng ngữ nghĩa; checks không bao giờ ✓ giả | GitHub review mặc định cho dev; ta thêm Summary tab cho Director không đọc hết diff |
| **Figma** | Canvas tool: toolbar trái, properties phải, layers; zoom/pan nhất quán; multiplayer cursor; comment pin trên canvas | Workflow Builder và Office Editor: cùng mô hình palette trái / inspector phải / canvas giữa; comment pin cho design approval `[Đề xuất]` | Figma phức tạp cho người mới — Workflow Builder cần template và validation dẫn dắt |

**Heuristic xuyên suốt rút ra:** (1) *Visibility of system status* bằng evidence thật; (2) *Match real world*: ẩn dụ "công ty" (department, approval) nhưng không ép ẩn dụ vào nơi làm giảm hiệu quả; (3) *User control*: Pause/Stop/Undo ở mọi nơi hợp lý; (4) *Consistency*: cùng state machine, cùng icon trên 22 màn; (5) *Error prevention*: confirm có nội dung, typed-confirm cho Emergency stop; (6) *Recognition over recall*: palette + tooltip + shortcut hint; (7) *Flexibility*: keyboard + mouse; (8) *Aesthetic & minimalist*: Office tối giản hoạt động trang trí; (9) *Help recover from errors*: lý do kẹt có cấu trúc + nút hành động; (10) *Help & docs*: shortcut help, onboarding checklist.

## 4. Phân tích UX Pixel Agents và my-virtual-office `[Đã xác minh]` từ README/screenshot

### 4.1 Pixel Agents (VS Code extension + CLI)

**Điểm mạnh UX:**
- **1 agent = 1 nhân vật**, animation gắn trực tiếp với hành động thật của agent (gõ khi edit file, đọc khi search, chạy lệnh) — đây là mô hình "evidence-driven animation" đúng tinh thần brief.
- **Speech bubble** khi agent chờ input / permission, kèm **chime tùy chọn** khi agent xong lượt hoặc xin quyền → attention routing tối giản nhưng hiệu quả.
- **Areas**: vẽ vùng đặt tên, map thư mục → agent mới ngồi trong vùng tương ứng → tiền thân của "department = area".
- Sub-agent/agent team hiển thị như nhân vật riêng với vai trò/lifecycle.
- Layout editor + import/export + asset pack.
- Trên điện thoại: thao tác tối thiểu (tap `+` để launch) → gợi ý mobile scope hẹp.
- Mô hình bảo mật token cho hành động mutation (client không token chỉ xem) → tương đồng viewer role.

**Điểm yếu / khoảng trống so với nhu cầu của ta:**
- Không có quản trị: không task board, không approval, không cost, không org — Pixel Agents là *visualizer của terminal*, không phải *control plane*.
- Thông tin ngoài hình ảnh nằm trong terminal → Director phải đọc terminal để hiểu chi tiết.
- Roadmap "Actually a game" (health bar, score, furniture that does things) — thú vị nhưng rủi ro gamification lệch mục tiêu (xem §6).

### 4.2 my-virtual-office (MVO)

**Điểm mạnh UX (từ screenshot và README):**
- Bố cục: canvas lớn bên trái + **dashboard panel phải** (PC performance, API usage, Branches/department với màu viền, Agent directory với trạng thái, Activity log) + **toolbar dưới** (Menu, Edit Office, Agents, Expand/Minimize All, Models, Cron, Reset). Đây là bố cục "canvas + side panel + bottom actions" hợp lý; ta giữ ý tưởng *HUD + Agent Panel* nhưng chuyển số liệu hệ thống ra Dashboard chuyên nghiệp.
- **Branch (department) có màu tường** → nhận diện nhóm nhanh trên canvas; trùng với chiến lược grouping theo department.
- Click agent → chat với markdown, tool activity feed, đính kèm; agent selector để chuyển nhanh.
- Office editor giàu tính năng (snap-to-grid, group save, layout library, import/export).
- Setup wizard `/setup` để kết nối harness trước.

**Điểm yếu / rủi ro nếu bắt chước:**
- Screenshot cho thấy **15 agent idle, 0 working** nhưng vẫn "watching TV / chatting / browsing" — hoạt động giả làm mờ ranh giới giữa *bận thật* và *trang trí*; Activity log toàn "chat bubbles minimized" (nhiễu, không có giá trị quyết định). Đây là bằng chứng trực tiếp cho rủi ro "game vs tool".
- Nhiều tính năng cảm xúc (pet, thời tiết, day/night, TV 5 kênh, ping-pong) làm tăng tải nhận thức và chi phí dev mà không tăng khả năng điều hành.
- Dashboard panel trộn số liệu máy (CPU/RAM) với số liệu agent; font pixel cho dữ liệu dạng bảng giảm khả năng đọc.
- Kiến trúc 1 file lớn, polling HTTP, không DB (theo research Agent 03) → không phải mẫu cho kiến trúc của ta; license AGPL → **không copy code/asset** `[Đã xác minh]`.
- Meeting "được kích hoạt bởi hoạt động agent hoặc lên lịch thủ công" — meeting giả lập không mang evidence.

### 4.3 Kết luận cho Virtual Office của ta
Giữ: nhân vật = agent, animation = tool event thật, bubble = cần người, Area = department, editor, panel bên phải, fullscreen. Bỏ: pet, TV, thời tiết, coffee, meeting giả, log nhiễu. Thêm: evidence progress trong panel, fallback list khi lỗi, reduced-motion, liên kết 1 click sang màn chuyên nghiệp.

## 5. Chiến lược tải nhận thức cho 18 agent `[Đề xuất]`

Giới hạn bộ nhớ làm việc ~4 ± 1 chunk; 18 agent × (state, task, progress, log) vượt xa. Chiến lược 5 lớp:

1. **Grouping theo department** (chunking): 18 → 4–6 nhóm; Office Area, Org Chart, filter mặc định, màu nhất quán per department (không per agent).
2. **Status-first views**: mọi danh sách sort mặc định theo mức cần chú ý (blocked/failed → waiting → working → idle → offline); bộ đếm trạng thái ở HUD/topbar thay cho đọc từng dòng.
3. **Progressive disclosure**: card (6 trường) → panel (12 trường + action) → detail (10 tab). Log chỉ hiện 1–2 dòng cuối cho tới khi mở Logs.
4. **Attention routing — "What needs me now"**: hàng đợi duy nhất gom approval.requested, agent.blocked, agent.failed, provider.critical, cost.threshold; xếp hạng theo severity × thời gian chờ × số task phụ thuộc; cho phép "Next" tuần tự (palette). Mọi kênh (toast, badge, bubble, bell) đều trỏ về cùng item để không tạo cảm giác "nhiều việc hơn thực tế".
5. **Giảm nhiễu động**: throttle feed (≤ 2 cập nhật/giây/agent), không toast cho `state_changed` thường, animation chỉ khi có event, reduced-motion, tắt sound mặc định, badge chỉ severity ≥ warning.

Chỉ số đo: thời gian từ mở app đến hành động đầu tiên đúng (target ≤ 20s với 3 item needs-you); tỷ lệ bỏ sót item blocked trong test (target 0).

## 6. Rủi ro usability của Virtual Office (game vs tool)

| Rủi ro | Biểu hiện | Giảm thiểu |
|---|---|---|
| **Ảo giác hoạt động** | Nhân vật "bận" khi agent idle (như MVO) → Director tin công việc đang tiến | Animation chỉ từ tool event; idle = ngồi yên/đi lại nhẹ; ghế trống khi offline |
| **Xao nhãng** | Nhìn nhân vật thay vì đọc Needs-you | Office không phải landing mặc định (`[Cần người dùng phê duyệt]`); HUD có Needs-you count; reduced-motion |
| **Thiếu thông tin** | Không biết agent làm gì cụ thể | Panel evidence + ticker + 1 click sang detail |
| **Khó mở rộng** | 18 nhân vật + nhãn chồng chéo trên màn nhỏ | Zoom semantic (ẩn nameplate khi zoom out, hiện bubble), minimap, filter làm mờ |
| **Nhận thức "đồ chơi"** | Khách hàng enterprise thấy thiếu nghiêm túc | Office là màn riêng, phần còn lại SaaS chuyên nghiệp `[Đã xác minh]`; chế độ "Professional sprites" tối giản `[Đề xuất]` |
| **Hiệu năng** | FPS thấp, CPU laptop nóng | Cap 60/30 FPS, dừng render khi tab ẩn |
| **Accessibility** | Canvas không đọc được bằng screen reader | Fallback list song song (DOM) có cùng dữ liệu; keyboard điều khiển camera và chọn nhân vật |
| **Gamification lệch** | Health bar/score (roadmap Pixel Agents) thúc đẩy hành vi tối ưu điểm | Không thêm score; chỉ hiển thị số liệu thật có ý nghĩa vận hành |

## 7. Khuyến nghị (ưu tiên)

1. **P0** Attention queue "Needs you now" dùng chung bởi Dashboard, Office HUD, mobile Home.
2. **P0** State machine agent + icon/màu thống nhất; legend ở Office; test color-blind.
3. **P0** Evidence progress `x/y` + tooltip nguồn ở mọi nơi; cấm % không mẫu số (đưa vào checklist QA).
4. **P0** Approval Center có Summary + Conversation + Diff theo comment cho revision.
5. **P1** Office tối giản hoạt động trang trí; fallback list; reduced-motion.
6. **P1** Lý do kẹt có cấu trúc (enum) để UI đưa nút hành động đúng.
7. **P1** Emergency stop: Pause all mặc định, Stop all typed-confirm, báo cáo per-agent.
8. **P2** Digest since last visit; Snooze approval.
9. **P2** Mobile monitoring + approvals; không authoring.
10. **P2** Office "professional sprite set" và theme đồng bộ Design System.

## 8. Kế hoạch usability test Phase 5/6

- **Phase 5 (sau khi 11 màn core chạy với provider thật):** moderated, 5 người (chủ sở hữu + 4 người có nền kỹ thuật, 2 trong số đó chưa dùng Claude Code), 60 phút/người, think-aloud, desktop.
  - Kịch bản: T1 Onboarding → agent đầu tiên (J1 bước 1–3); T2 Tạo project từ prompt và tìm agent đang làm gì trong Office (J1 4–5); T3 Xử lý 3 item Needs-you gồm 1 approval, 1 blocked, 1 provider lỗi (J2/J3); T4 Review PR với 20 file, 2 test fail, 1 finding (J1 9); T5 Emergency stop rồi resume 2 agent (J4); T6 Dạy skill và gắn (J5).
  - Đo: task success, time-on-task, số lỗi, SEQ (1–7) sau mỗi task, SUS cuối; log click path; quan sát "nhìn Office vs Dashboard".
  - Tiêu chí đạt: ≥ 80% task success không trợ giúp; SEQ trung bình ≥ 5.5; không bỏ sót blocked agent; SUS ≥ 72.
- **Phase 6 (polish):** unmoderated remote 8–10 người, 3 kịch bản ngắn (approval mobile, tìm agent qua palette, đọc cost), A/B landing Dashboard vs Office (đo thời gian tới hành động đầu tiên), kiểm thử accessibility (keyboard-only, screen reader cho fallback list, contrast), kiểm thử tải nhận thức với 18 agent mô phỏng từ **replay log thật** (không dữ liệu giả hiển thị như thật — ghi rõ "replay").
- **Sản phẩm đầu ra:** báo cáo findings theo severity (blocker/major/minor), video clip, danh sách thay đổi gắn màn hình, cập nhật `USABILITY_AUDIT_CHECKLIST.md`.

## 9. Quyết định & lý do

| Quyết định | Lý do | Nhãn |
|---|---|---|
| Học pattern, không sao chép UI/code từ tham chiếu | Khác biệt sản phẩm; license AGPL của MVO; asset nhân vật của Pixel Agents thuộc bên thứ ba | `[Đã xác minh]` (license) |
| Office tối giản hoạt động trang trí | Bằng chứng screenshot MVO về ảo giác hoạt động | `[Cần người dùng phê duyệt]` |
| Attention queue là trục chính của IA | Giải quyết trực tiếp tải nhận thức 18 agent | `[Đề xuất]` |
| Usability test với provider thật, không mock | Nguyên tắc evidence-driven áp dụng cả cho test | `[Đề xuất]` |

## 10. Dependencies
- Agent 11: legend/status token, reduced-motion guideline, chart style.
- Agent Backend: enum blocked_reason, attention endpoint, replay log cho test tải.
- Agent Office: sprite set tự vẽ, semantic zoom, fallback DOM list.
- Chủ sở hữu: tham gia test Phase 5, cung cấp 4 người tham gia.

## 11. Rủi ro
- Nhận định về sản phẩm tham chiếu (Linear… Figma) chưa được xác minh lại trong phiên → có thể lệch với phiên bản hiện tại; chỉ dùng ở mức pattern.
- Không có người dùng thật ngoài chủ sở hữu → bias; cần tuyển ít nhất 4 người ngoài.
- Quyết định tối giản Office có thể giảm "niềm vui" là giá trị cảm xúc chủ sở hữu mong muốn.

## 12. Vấn đề chưa giải quyết
1. Mức "vui" chấp nhận được của Office (idle animation nào được phép)?
2. Landing mặc định (A/B ở Phase 6 hay chốt sớm)?
3. Có cần nghiên cứu thêm với người dùng enterprise (viewer) trước Phase 4?

## 13. Acceptance Criteria
- [ ] Mỗi khuyến nghị P0 được ánh xạ tới ít nhất một mục trong `SCREEN_SPECIFICATION.md` và `USABILITY_AUDIT_CHECKLIST.md`.
- [ ] Kế hoạch test Phase 5 có kịch bản, thước đo, tiêu chí đạt (mục 8) và được chủ sở hữu phê duyệt lịch.
- [ ] Không có code/asset nào từ repo tham chiếu được đưa vào sản phẩm (kiểm tra ở Phase 2 code review).
- [ ] Rủi ro Office §6 có mục giảm thiểu tương ứng trong spec Office.
