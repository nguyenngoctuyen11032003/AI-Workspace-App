# HERDR_DESIGN_INTEGRATION.md — Áp dụng thiết kế kiểu herdr vào AI WORKSPACE

Ngày: 2026-10-30. Cơ sở phân tích: `docs/research/HERDR_ANALYSIS.md`. Tài liệu này là **phụ lục sửa đổi** cho các tài liệu trong `docs/design/`. Khi có mâu thuẫn, tài liệu này thắng cho các hạng mục liệt kê dưới đây, cho đến khi chủ sở hữu duyệt và các tài liệu gốc được hợp nhất.

Nhãn: **[ĐXuất]** đề xuất; **[GĐ]** giả định; **[PD]** cần người dùng phê duyệt.

## 1. Mục đích và nguyên tắc
Đưa vào những gì khiến herdr trông chuyên nghiệp và dễ điều hành nhiều agent: nền phẳng, một màu nhấn, nhãn mờ viết thường, trạng thái theo nhu cầu chú ý, sidebar tổng hợp. Giữ nguyên các ràng buộc của Prompt 3: premium SaaS, dark/light/system, không gradient hay glow, Virtual Office là nơi duy nhất mang phong cách game.

Ba nguyên tắc rút ra:
1. **Chú ý trước, kỹ thuật sau.** Người điều hành 18 agent phải thấy ngay cái gì cần mình, trước khi thấy cái gì đang chạy.
2. **Phân tầng bằng độ sáng nền, không bằng viền và bóng.** Giảm "card trong card".
3. **Một màu nhấn cho hành động và focus, màu trạng thái chỉ là chấm hoặc chữ nhỏ.**

## 2. Attention state: lớp trạng thái hiển thị cho người điều hành [ĐXuất]
12 lifecycle state (Created…Disabled) vẫn là trạng thái nghiệp vụ trong database. Bên trên thêm **5 attention state** tính toán, dùng cho Agent Rail, Dashboard, huy hiệu và thông báo.

| Attention | Điều kiện suy ra từ dữ liệu thật | Màu token | Hành vi |
|---|---|---|---|
| `blocked` | Có approval đang chờ, hoặc agent `Blocked`/`Waiting` cần input, hoặc lỗi cần quyết định | `--status-blocked` | Lên đầu danh sách, huy hiệu đỏ, toast |
| `working` | Agent `Running` và có event trong cửa sổ TTL | `--status-running` | Chấm đặc, thời gian chạy |
| `done` | Task hoàn thành có bằng chứng, **người dùng chưa xem** | `--status-info` (xanh dương) | Chấm xanh, tắt khi mở chi tiết |
| `idle` | Không có việc, hoặc xong và đã xem | `--status-idle` | Chữ mờ |
| `unknown` | Mất heartbeat, provider không báo, chưa phân loại được | `--text-muted` | Cảnh báo nhẹ, không giả định |

Quy tắc:
- **Mỗi client tự theo dõi cái đã xem** (bảng `notification_reads` hoặc cột `last_seen_event_seq` theo `user + client`). Xem ở một cửa sổ không xóa huy hiệu ở cửa sổ khác. Đề xuất bổ sung cho `DATABASE_DESIGN.md` [PD].
- Trạng thái project và department là **tổng hợp** theo thứ tự ưu tiên `blocked > working > done > idle`, kèm số lượng.
- `unknown` là trạng thái hợp lệ. Không bao giờ hiển thị `working` khi không có event thật (khớp luật "không giả lập tiến độ").
- Ánh xạ từ lifecycle sang attention là hàm thuần, có test, nằm trong package dùng chung.

## 3. Agent Rail (sidebar kiểu hai vùng) [ĐXuất]
Bổ sung vào `NAVIGATION_STRUCTURE.md`: sidebar điều hướng chính giữ nguyên các nhóm đã duyệt. Thêm **Agent Rail** là một panel hai vùng, mở/đóng được, dùng được ở mọi màn hình.

```
┌ Rail ────────────────────┐
│ projects                 │
│ ● aiw-core      main  ▲2 │  ← chấm = attention tổng hợp, nhánh git, ahead/behind
│ ◐ billing-api   feat/x   │
│ ○ docs-site     main     │
│ ── kéo để đổi tỉ lệ ──── │
│ agents                18 │
│ ⚠ backend-dev  blocked   │  ← blocked lên đầu
│ ● architect    working   │
│ ● frontend-dev working   │
│ ✓ ba-analyst   done      │
│ ○ pm           idle      │
└──────────────────────────┘
```
- Vùng trên: project, mỗi dòng gồm chấm attention, tên, nhánh git, ahead/behind. Vùng dưới: agent, nhãn trạng thái căn phải, sắp theo ưu tiên attention rồi theo phòng ban.
- Thanh chia kéo được, tỉ lệ nhớ theo người dùng (`ui_preferences`). Mỗi vùng tối thiểu 3 dòng.
- **Dòng cấu hình bằng token** (`state_icon`, `project`, `git_status`, `current_task`, `provider`, `runtime`, `cost`). Người dùng chọn token hiển thị, chế độ gọn mặc định chỉ có chấm, tên, trạng thái. Có thể khác nhau theo provider.
- Nhấp một agent: mở panel chi tiết bên phải (không rời trang hiện tại). Nhấp project: đổi project scope.
- Trên mobile: Rail thành tab "Agents" ở bottom nav, đúng với phạm vi mobile giám sát và phê duyệt.
- Với 18 agent: nhóm theo phòng ban có thể thu gọn, chỉ hiện nhóm có agent `blocked` hoặc `working` khi bật bộ lọc "cần chú ý".

## 4. Token màu và theme [ĐXuất]
Ánh xạ palette herdr sang token của chúng ta (xem `DESIGN_TOKENS.md`). Chỉ **thêm** token thiếu, không đổi tên token đã có.

| Token herdr | Token AI WORKSPACE | Ghi chú |
|---|---|---|
| `panel_bg` | `--surface-overlay` | Menu, popover, dialog |
| `sidebar_bg` | `--surface-rail` (mới) | Cho phép trong suốt hoặc khác nền chính một bậc |
| `active_row_bg` | `--row-active` (mới) | Dòng đang chọn |
| `selection_bg` | `--row-cursor` (mới) | Con trỏ phím |
| `surface0 / surface1 / surface_dim` | `--surface-2 / --surface-3 / --border-subtle` | Hover, active, đường phân cách |
| `overlay0 / overlay1 / subtext0` | `--text-muted / --text-secondary` | Chữ phụ |
| `green / yellow / red / blue / peach` | `--status-completed / --status-running / --status-blocked / --status-info / --status-warning` | Giữ giá trị riêng đã duyệt trong tài liệu token |
| `mauve` | `--text-meta` (mới) | Tên nhánh, nhãn đặc biệt |
| `accent` | `--accent` (hành động) | **Khác** `--ai-accent` tím dành cho thành phần AI (D-015) |

Bổ sung:
- **Theme preset**: Mặc định "AIW Dark" và "AIW Light" theo `DESIGN_TOKENS.md`. Thêm tùy chọn preset kiểu Catppuccin Mocha/Latte, Tokyo Night, Nord, Vesper (giá trị tham khảo từ herdr, MIT-compatible dưới Apache-2.0 kèm attribution) ở Settings → Appearance. Chủ sở hữu quyết định có bật hay không [PD].
- `auto_switch`: theo `prefers-color-scheme` (đã có trong yêu cầu "System Theme").
- **Ghi đè token tùy biến** (`theme.custom`) lưu theo người dùng, kiểm tra hợp lệ và cảnh báo token lạ như herdr (`unknown theme.custom key`).
- **Màu chữ tương phản tự động** trên nền trạng thái: tính theo độ tương phản WCAG, test đảm bảo AA cho mọi cặp trạng thái.
- Phong cách chữ: nhãn phụ viết thường, mờ; tên đối tượng chọn dùng chữ chính. Font mono cho ID, nhánh, log; UI chính giữ Inter (D-015).

## 5. Độ bền phiên và phục hồi giao diện [ĐXuất]
Áp dụng bảng "cái gì còn sống" của herdr cho yêu cầu Reliability nhóm G:

| Tình huống | Tiến trình agent | Trạng thái workflow/task | UI |
|---|---|---|---|
| Đóng tab hoặc client | Tiếp tục (backend độc lập) | Giữ | Mở lại thấy ngay, replay từ `lastEventId` |
| Backend restart | Mất tiến trình | Phục hồi từ Postgres, task `RUNNING` đánh dấu `RECOVERING` rồi resume bằng session id nếu provider hỗ trợ | Báo rõ "đã phục hồi từ checkpoint" |
| Provider không resume được | n/a | Task về `READY` với attempt+1, ghi lý do | Hiển thị cảnh báo |

Bổ sung:
- **Snapshot bố cục Office và bố cục Rail**: giữ tối đa 48 snapshot, chụp không quá một lần mỗi khoảng cố định.
- **Giữ bản hỏng**: khi dữ liệu bố cục không đọc được hoặc phiên bản mới hơn, sao lưu nguyên bản gốc trước khi ghi đè, giữ 3 bản gần nhất.
- Thư mục workspace biến mất: task giữ nguyên trong danh sách kèm lỗi, không biến mất hay chuyển sang thư mục khác.
- **Lịch sử màn hình/terminal tắt mặc định** vì có thể chứa secret. Bật theo từng project kèm cảnh báo, đi qua bộ lọc redaction.
- Lệnh resume do agent báo (GenericCliAdapter) phải qua kiểm tra: tên lệnh trần trong `PATH`, tối đa 64 đối số và 8 KiB, không có dấu nháy đơn hay ký tự điều khiển.

## 6. Điều khiển bởi agent: biến môi trường, CLI và skill [ĐXuất, Phase 3]
- Mỗi tiến trình agent nhận `AIW_ENV=1`, `AIW_AGENT_ID`, `AIW_TASK_ID`, `AIW_PROJECT_ID`, `AIW_API_URL`. Token API ngắn hạn **riêng cho mỗi agent**, giới hạn theo quyền của agent đó, không chia sẻ giữa các agent (khớp luật cách ly credential). Chặn khởi chạy lồng nhau nếu `AIW_ENV=1`.
- CLI `aiw` trả JSON: `aiw task show`, `aiw task handoff`, `aiw request-help`, `aiw request-review`, `aiw wait --agent <id> --state blocked`. Mọi lệnh đi qua Message Bus và Approval có sẵn, không đường tắt.
- Skill "aiw-agent" theo chuẩn Agent Skills (xem `SKILL_SYSTEM.md`) dạy agent: kiểm tra `AIW_ENV` trước, lấy ID từ phản hồi JSON thay vì đoán, không sửa file ngoài manifest.
- Báo cáo từ hook/provider mang `source` ổn định và `seq` tăng đơn điệu theo từng `source`. Server bỏ báo cáo có `seq` không lớn hơn báo cáo trước (chống lặp và đảo thứ tự). Thêm vào `EVENT_SYSTEM.md`.
- Xuất **JSON Schema của event và REST** từ chính backend (`aiw api schema`), đồng bộ với OpenAPI.

## 7. Terminal sống của agent trong Code Workspace [Đề xuất, hoãn, PD]
herdr mạnh nhờ pane là terminal thật. Với agent chạy bằng SDK chúng ta không có TTY, nên thay bằng **Activity Stream** có cấu trúc (tool call, diff, log) đã có trong thiết kế. Phương án bổ sung cho Phase 7:
- Chế độ "Interactive session": chạy Claude Code/Codex dạng TUI trong PTY (node-pty trên Windows qua ConPTY) và hiển thị bằng xterm.js, hữu ích khi người dùng muốn trực tiếp tiếp quản.
- Hoặc dùng **HerdrRuntime** (xem `HERDR_ANALYSIS.md` mục 8.2) để người dùng gắn vào pane herdr đã có.
Quyết định cần PoC và phê duyệt vì làm yếu cổng phê duyệt có cấu trúc.

## 8. Phím tắt và chế độ nhập [ĐXuất]
- Command Palette (`Ctrl/Cmd+K`) là điểm vào chính. Thêm "mode điều hướng": phím mũi tên và `j/k` di chuyển trong Rail và danh sách task, `Enter` mở, `.` mở menu hành động, `Esc` thoát. Tránh xung đột phím trình duyệt (herdr có bảng phím bị hệ điều hành chiếm).
- Mọi hành động đạt được bằng chuột **và** bàn phím, hiển thị phím tắt trong tooltip và menu chuột phải.

## 9. Thay đổi cần ghi vào tài liệu khác
| Tài liệu | Thay đổi đề xuất |
|---|---|
| `design/NAVIGATION_STRUCTURE.md` | Thêm Agent Rail (mục 3) |
| `design/DESIGN_TOKENS.md` | Thêm token `--surface-rail`, `--row-active`, `--row-cursor`, `--text-meta`, bảng attention |
| `design/DESIGN_SYSTEM.md` | Thêm component `AgentRail`, `AttentionDot`, `RailRow` (props theo token) |
| `design/SCREEN_SPECIFICATION.md` | Dashboard và Agent Management dùng attention state, nhóm "cần chú ý" lên đầu |
| `design/INTERACTION_DESIGN.md` | Mode điều hướng bàn phím (mục 8), tắt huy hiệu `done` khi đã xem |
| `DATABASE_DESIGN.md` | Bảng theo dõi đã xem theo client, `ui_preferences` (tỉ lệ Rail, token hiển thị, theme custom) |
| `EVENT_SYSTEM.md` | `source` + `seq` đơn điệu, ánh xạ lifecycle → attention |
| `architecture/AGENT_COLLABORATION.md` | CLI `aiw` và biến môi trường `AIW_*` |
| `research/LICENSE_AND_REUSE.md`, THIRD_PARTY_NOTICES | Thêm herdr (Apache-2.0) |

## 10. Rủi ro
- Quá nhiều lớp trạng thái gây khó hiểu cho người mới. Giảm thiểu: Rail và badge chỉ dùng attention, giải thích bằng tooltip, USABILITY_AUDIT kiểm tra.
- Preset theme bên thứ ba làm lệch nhận diện thương hiệu. Giảm thiểu: mặc định vẫn là AIW Dark/Light.
- CLI `aiw` mở thêm bề mặt tấn công. Giảm thiểu: token ngắn hạn theo agent, đi qua cùng policy engine, test bảo mật.

## 11. Vấn đề chưa giải quyết
1. Bật preset theme bên thứ ba hay chỉ AIW Dark/Light. [PD]
2. Có làm chế độ Interactive session hoặc HerdrRuntime không, và khi nào. [PD]
3. Mật độ mặc định của Rail: chế độ gọn hay đầy đủ token.

## 12. Acceptance Criteria
- [ ] Hàm lifecycle → attention có test đủ 12 trạng thái và các điều kiện `unknown`.
- [ ] Rail hiển thị 18 agent, blocked luôn ở trên cùng, tổng hợp project đúng ưu tiên.
- [ ] Mở agent ở một client không xóa huy hiệu `done` ở client khác (test hai client).
- [ ] Mọi cặp màu trạng thái đạt tương phản AA ở cả hai theme.
- [ ] `seq` thấp hơn hoặc bằng bị bỏ qua và không đổi trạng thái (test).
- [ ] Phục hồi bố cục từ snapshot và giữ bản hỏng có test.
