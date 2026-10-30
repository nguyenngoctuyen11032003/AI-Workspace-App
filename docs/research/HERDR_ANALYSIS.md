# HERDR_ANALYSIS.md — Phân tích herdr làm tham chiếu thiết kế

Ngày: 2026-10-30. Nguồn: `D:\Code\Tuyen\Agent management\herdr` (clone của https://github.com/herdrdev/herdr, commit `ee5bfb50`, Apache-2.0). Chủ sở hữu yêu cầu: xem phần thiết kế, đánh giá và kết hợp vào AI WORKSPACE.

Nhãn: **[ĐX]** Đã xác minh từ mã nguồn hoặc tài liệu trong repo; **[ĐXuất]** Đề xuất; **[GĐ]** Giả định; **[PD]** Cần người dùng phê duyệt.

## 1. Mục đích và phạm vi
Xác định những gì herdr làm tốt về thiết kế giao diện, mô hình trạng thái agent, độ bền phiên làm việc và điều khiển bằng agent, rồi quyết định phần nào áp dụng cho AI WORKSPACE. Không phải bản phân tích toàn bộ mã Rust. Phạm vi đọc: README, `src/ui/`, `src/app/state.rs` (Palette), `src/config/theme.rs`, `src/detect/manifests/`, tài liệu `docs/next/website/src/content/docs/` (concepts, session-state, socket-api, add-herdr-support, agent-skill), `skills/herdr/SKILL.md`, `assets/screenshot.png`.

## 2. herdr là gì [ĐX]
- Một **runtime terminal cho coding agent**, viết bằng Rust, một binary, chạy trong terminal có sẵn (ratatui). Không phải ứng dụng web hay Electron.
- Kiến trúc **server nền + client gắn vào**. Server giữ pane và tiến trình. Client là giao diện terminal. Đóng client không dừng agent (`ctrl+b q` để detach).
- Mô hình đối tượng: **workspace** (một dự án) → **tab** (một bố cục) → **pane** (một terminal thật). Agent là tiến trình được nhận diện bên trong pane.
- Nhận diện agent qua tiến trình foreground, manifest màn hình và integration tùy chọn. Hỗ trợ 22 agent có manifest (claude, codex, cursor, opencode, gemini, grok, kimi, hermes và các agent khác).
- Điều khiển bằng CLI và socket API cục bộ. Agent có thể tự điều khiển herdr bằng skill (`skills/herdr/SKILL.md`).

herdr **không** là công cụ điều phối công ty phần mềm. Không có PM agent, workflow, approval có cấu trúc hay database. Giá trị của nó với chúng ta là **mẫu thiết kế vận hành**, không phải logic nghiệp vụ.

## 3. Nhận xét thiết kế giao diện [ĐX từ screenshot và mã]

### 3.1 Ngôn ngữ thị giác
- Nền tối than chì, gần như một lớp phẳng. Phân tầng bằng **độ sáng nền** (panel, active row, selection, surface) chứ không bằng viền dày hay đổ bóng.
- Chữ nhãn viết thường, màu mờ (`overlay0`, `subtext0`). Chỉ tên đối tượng đang chọn dùng chữ sáng (`text`).
- **Một màu nhấn duy nhất** cho focus (viền pane đang chọn). Màu trạng thái chỉ xuất hiện dưới dạng chấm hoặc chữ nhỏ.
- Font monospace xuyên suốt. Mật độ thông tin cao nhưng không rối nhờ khoảng trắng dọc và căn lề cố định.

### 3.2 Bộ token màu (struct `Palette` trong `src/app/state.rs`) [ĐX]
| Token | Vai trò |
|---|---|
| `accent` | Highlight, viền đang focus |
| `panel_bg` | Nền thanh tab, panel nổi, overlay, modal |
| `sidebar_bg` | Nền sidebar (có thể để trong suốt) |
| `active_row_bg` | Dòng workspace/agent đang chọn |
| `selection_bg` | Con trỏ ở chế độ điều hướng |
| `surface0`, `surface1`, `surface_dim` | Bề mặt chọn, hover/active, đường phân cách |
| `overlay0`, `overlay1` | Chữ phụ, số |
| `text`, `subtext0` | Chữ chính, chữ mờ |
| `green` | Done, idle |
| `yellow` | Working, running |
| `red` | Cần chú ý, blocked |
| `blue` | Chưa xem, thông báo hoàn thành |
| `teal` | Dấu thông báo |
| `peach` | Bị ngắt, cảnh báo |
| `mauve` | Tên nhánh, nhãn đặc biệt |

Giá trị mẫu (Catppuccin Mocha, mặc định): `panel_bg #181825`, `active_row_bg #1E1E2E`, `selection_bg #313244`, `surface1 #45475A`, `overlay0 #6C7086`, `text #CDD6F4`, `green #A6E3A1`, `yellow #F9E2AF`, `red #F38BA8`, `blue #89B4FA`, `mauve #CBA6F7`, `peach #FAB387`. Bản sáng (Latte): `panel_bg #EFF1F5`, `text #4C4F69`, `accent #1E66F5`.

Có 18 theme dựng sẵn (catppuccin, tokyo-night, dracula, nord, gruvbox, one-dark, solarized, kanagawa, rose-pine, vesper và bản sáng tương ứng), `auto_switch` theo giao diện sáng/tối của hệ điều hành, và `theme.custom` ghi đè từng token. Có hàm `panel_contrast_fg` tính màu chữ tương phản trên nền màu trạng thái.

### 3.3 Bố cục
- Sidebar chia **hai vùng có thể kéo tỉ lệ**: trên là danh sách **workspace** (tên, nhánh git, chấm trạng thái), dưới là danh sách **agents** (tên agent, nhãn trạng thái căn phải: `idle`, `working`). Số dòng tối thiểu mỗi vùng là 3.
- Mỗi dòng cấu hình được bằng **token** (`state_icon`, `workspace`, `git_status`, `$summary`, `terminal_title_stripped`, token tùy biến), cho phép khác nhau theo từng loại agent.
- Vùng nội dung là các pane chia ngang/dọc. Pane đang focus có viền màu nhấn.
- Thanh tab mảnh ở trên, thanh trạng thái dưới. Toast xác nhận nhỏ, vị trí cấu hình được (6 vị trí).
- Chuột và bàn phím đều là công dân hạng nhất. Có ba chế độ: terminal, prefix, navigate.

## 4. Mô hình trạng thái agent [ĐX]
Năm trạng thái, định nghĩa theo **nhu cầu chú ý của con người** chứ không theo vòng đời kỹ thuật:

| Trạng thái | Ý nghĩa |
|---|---|
| `blocked` | Cần input, phê duyệt hoặc quyết định |
| `working` | Đang chạy |
| `done` | Vừa xong, người dùng chưa xem |
| `idle` | Xong hoặc đang chờ, đã xem |
| `unknown` | Không phân loại được chắc chắn |

Điểm đáng giá:
1. Trạng thái **`done` chưa xem** tách khỏi `idle`. Mỗi client theo dõi riêng những hoàn thành nó đã hiển thị. Đây là cơ chế "không cần đi tìm agent bị kẹt".
2. Trạng thái workspace là **tổng hợp từ các agent** bên trong, nên sidebar cho biết dự án nào cần chú ý.
3. Lưu ý trong tài liệu: `idle` và `done` không bảo đảm rằng một hộp thoại phê duyệt lạ đã được nhận diện. Nghĩa là **nhận diện qua màn hình là heuristic**, không phải bằng chứng.
4. Giao thức báo trạng thái từ chính agent (`pane report-agent`) dùng `--source` ổn định theo từng integration và `--seq` **tăng đơn điệu**, herdr bỏ qua báo cáo có seq không lớn hơn. Chống lặp và chống đảo thứ tự.

## 5. Nhận diện agent bằng manifest khai báo [ĐX]
`src/detect/manifests/*.toml` mô tả quy tắc theo dữ liệu, không hardcode. Mỗi quy tắc có `id`, `state`, `priority`, `region` (ví dụ `osc_title`, `bottom_non_empty_lines(12)`), `regex`/`line_regex`/`contains`/`any`/`not`. Manifest có `version`, `min_engine_version`, `updated_at` và có cơ chế cập nhật riêng (`manifest_update.rs`). Quy tắc có điều kiện loại trừ (`not`) để chữ người dùng gõ không giả mạo tín hiệu.

Bài học: heuristic màn hình **mong manh theo phiên bản CLI** và cần cập nhật thường xuyên, nên herdr tách thành dữ liệu có phiên bản.

## 6. Độ bền phiên và phục hồi [ĐX]
| Tình huống | Tiến trình còn chạy | Bố cục trở lại | Hội thoại agent tiếp tục |
|---|---|---|---|
| Detach rồi attach lại | Có | Có | Có |
| Server restart | Không | Có | Chỉ với native resume |
| Update có handoff | Cố gắng tối đa | Có | Có nếu handoff thành công |

- Snapshot restore khôi phục **hình dạng** (workspace, tab, pane, cwd, focus), không khôi phục tiến trình.
- `session.json` hỏng: giữ nguyên bản gốc trong `session-backups/` (3 bản gần nhất) trước khi ghi đè. Giữ tới 48 snapshot bố cục trong `session-snapshots/`.
- Lịch sử màn hình pane **tắt mặc định** vì có thể chứa secret. Chỉ lưu khi bật `experimental.pane_history`.
- Native agent session restore: agent báo lệnh resume, herdr chạy lại lệnh đó trong cùng thư mục. Ràng buộc chặt: lệnh là tên trần trong `PATH`, tối đa 64 đối số và 8 KiB, không có dấu nháy đơn hay ký tự điều khiển. Có `resume_not_accepted` và `invalid_resume_argv`.
- Thư mục lưu bị mất: pane vẫn nằm trong bố cục kèm lỗi, không biến mất hay chuyển về home.

## 7. Điều khiển bởi agent [ĐX]
- Mỗi tiến trình trong pane nhận `HERDR_ENV=1`, `HERDR_PANE_ID`, `HERDR_BIN_PATH`, `HERDR_SOCKET_PATH`, `HERDR_WORKSPACE_ID`, `HERDR_TAB_ID`. Có lớp chống lồng nhau (`HERDR_ENV=1` chặn khởi chạy đệ quy).
- Ba lớp tích hợp dùng chung một bề mặt: agent skill, CLI (trả JSON), socket API (có subscription sự kiện). `herdr api schema --json` xuất JSON Schema của giao thức.
- `herdr agent wait` chặn cho đến khi agent khác thật sự `blocked`. `agent start` không tự tạo hay di chuyển bố cục.
- Skill của herdr dặn agent: kiểm tra `HERDR_ENV` trước, lấy ID từ phản hồi JSON thay vì đoán, ID đóng không tái sử dụng.

## 8. Ý nghĩa với AI WORKSPACE
Điểm khác biệt cơ bản: herdr **quan sát terminal** của agent chạy sẵn. AI WORKSPACE **sở hữu phiên** qua Agent SDK (D-004) hoặc `codex app-server` (D-005), nên có sự kiện có cấu trúc, đáng tin hơn heuristic màn hình. Vì vậy chúng ta lấy **mẫu thiết kế và mô hình trạng thái**, không lấy cơ chế nhận diện màn hình làm đường chính.

### 8.1 Bảng quyết định tái sử dụng
| Hạng mục | Nguồn trong herdr | Quyết định | Mức |
|---|---|---|---|
| Mô hình 5 trạng thái chú ý (blocked/working/done/idle/unknown) và `done` chưa xem theo từng client | `docs/.../concepts.mdx` | **Áp dụng** làm lớp "attention state" bên trên 12 lifecycle state | [ĐXuất] |
| Tổng hợp trạng thái từ agent lên project/department | `src/ui/sidebar.rs` | **Áp dụng** | [ĐXuất] |
| Sidebar hai vùng (projects trên, agents dưới) kéo tỉ lệ, dòng cấu hình bằng token | `src/ui/sidebar*`, `src/config/sidebar` | **Áp dụng** cho Agent Rail của shell | [ĐXuất] |
| Palette 20 token và cơ chế `theme.custom`, `auto_switch` | `src/app/state.rs`, `src/config/theme.rs` | **Áp dụng** cấu trúc token và preset; giữ accent AI tím riêng (D-015) | [ĐXuất] |
| Hàm màu chữ tương phản trên nền trạng thái | `src/ui/widgets.rs` | **Áp dụng** ý tưởng, tính theo WCAG | [ĐXuất] |
| Seq tăng đơn điệu + source ổn định cho báo cáo trạng thái | `add-herdr-support.mdx` | **Áp dụng** cho Provider event ingest và hook | [ĐXuất] |
| Biến môi trường tiêm vào tiến trình agent, chống lồng nhau | `skills/herdr/SKILL.md` | **Áp dụng** với tiền tố `AIW_` | [ĐXuất] |
| CLI `aiw` trả JSON + skill cho agent (`wait` đến khi blocked) | `socket-api.mdx` | **Áp dụng**, Phase 3 | [ĐXuất] |
| JSON Schema giao thức xuất từ binary | `herdr api schema` | **Áp dụng** cho event schema và OpenAPI | [ĐXuất] |
| Snapshot bố cục, giữ bản hỏng trước khi ghi đè, tắt lịch sử màn hình mặc định | `session-state.mdx` | **Áp dụng** cho phục hồi Office layout và workflow | [ĐXuất] |
| Ràng buộc lệnh resume (tên trần, giới hạn đối số, cấm nháy đơn) | `add-herdr-support.mdx` | **Áp dụng** cho resume của GenericCliAdapter | [ĐXuất] |
| Manifest nhận diện dạng dữ liệu có version | `src/detect/manifests/*.toml` | **Tham khảo**, dùng cho GenericCliAdapter và agent ngoài hệ thống. Không dùng cho Claude/Codex được quản lý | [ĐXuất] |
| Chế độ chuột + phím, prefix/navigate | `docs/.../keyboard.mdx` | **Tham khảo** cho Command Palette và phím tắt | [ĐXuất] |
| Server nền + client gắn vào, detach không dừng việc | `README.md` | **Đã nằm trong thiết kế** (backend độc lập UI). Xác nhận lại | [ĐX] |
| Nhiều máy trong một cửa sổ | `connecting-machines.mdx` | **Hoãn** Phase 7 | [GĐ] |
| Mã Rust, crate `ghostty-vt`, renderer ratatui | `src/`, `crates/` | **Không dùng**. Khác ngôn ngữ và khác nền tảng | [ĐX] |
| Âm thanh thông báo | `src/sound.rs`, `assets/sounds` | **Tham khảo**. Kiểm tra license asset trước khi dùng file âm thanh | [PD] |

### 8.2 Cơ hội đặc thù: herdr là runtime tùy chọn cho agent
Trên máy này đã có `herdr 0.9.5` với một session mặc định đang chạy và socket tại `C:\Users\TUYEN\AppData\Roaming\herdr\herdr.sock` [ĐX]. Đề xuất một **HerdrRuntime** tùy chọn ở Phase 2 hoặc 7: AI WORKSPACE chạy Claude Code/Codex dạng TUI trong pane herdr, đọc trạng thái `blocked/working/done` qua socket API, người dùng có thể gắn vào xem terminal thật của từng agent. Ưu điểm: giao diện terminal sống miễn phí, bền vững khi UI đóng. Nhược điểm: mất `canUseTool` có cấu trúc, trạng thái dựa vào heuristic, herdr Windows còn beta. **Không** thay thế đường SDK. Cần PoC trước khi quyết định [GĐ] [PD].

## 9. License [ĐX]
Apache-2.0. Cho phép dùng, sửa, phân phối kể cả thương mại, với điều kiện giữ bản license, ghi chú thay đổi trên file đã sửa và giữ NOTICE nếu có (repo không có file NOTICE ở thư mục gốc). Dữ liệu manifest TOML và văn bản tài liệu dùng lại được kèm attribution. Chưa kiểm tra giấy phép riêng của thư mục `vendor/`, `assets/sounds` và font nếu có, nên **không** sao chép các phần đó khi chưa kiểm tra [PD]. Không dùng logo và tên herdr cho sản phẩm.

## 10. Rủi ro
| Rủi ro | Giảm thiểu |
|---|---|
| Ép phong cách terminal lên giao diện web làm mất tính "premium SaaS" đã yêu cầu ở Prompt 3 | Chỉ lấy nguyên tắc (phẳng, một màu nhấn, nhãn mờ, mật độ cao). Giữ chuẩn Linear/Vercel cho phần còn lại |
| Nhầm heuristic màn hình với bằng chứng | Trạng thái chính lấy từ sự kiện provider có cấu trúc. Manifest chỉ cho agent ngoài |
| Quá nhiều trạng thái làm rối (12 lifecycle + 5 attention) | Hiển thị chỉ attention state ở Rail, lifecycle chi tiết ở Agent Detail |
| herdr Windows còn beta | HerdrRuntime chỉ là tùy chọn sau PoC |

## 11. Vấn đề chưa giải quyết
1. `vendor/` và `assets/sounds` chưa kiểm tra license.
2. Chưa chạy thử socket API của herdr đang chạy trên máy để xác minh độ ổn định trên Windows.
3. Có nên bật chế độ "terminal pane" cho từng agent trong Code Workspace (xterm.js + node-pty) hay không, xem `docs/design/HERDR_DESIGN_INTEGRATION.md` mục 6.

## 12. Acceptance Criteria
- [ ] Mỗi mục "Áp dụng" ở 8.1 có chỗ tương ứng trong tài liệu thiết kế (xem `HERDR_DESIGN_INTEGRATION.md`).
- [ ] THIRD_PARTY_NOTICES có mục herdr (Apache-2.0) trước khi bất kỳ dữ liệu manifest nào được đưa vào.
- [ ] Kiểm tra license `vendor/` và `assets/sounds` trước khi dùng.
