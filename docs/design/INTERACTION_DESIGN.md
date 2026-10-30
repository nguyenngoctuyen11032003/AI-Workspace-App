# INTERACTION DESIGN — MICRO-INTERACTIONS, MOTION & KEYBOARD

> Phase 0 · Agent 11 · Phiên bản 0.1 · Phụ thuộc `DESIGN_TOKENS.md` §3.8 (motion tokens) và `DESIGN_SYSTEM.md`.
> Đánh dấu: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**

## 1. Mục đích

Quy định cách UI phản hồi hành động người dùng và sự kiện real-time từ tối đa 18 agent, sao cho "purposeful motion only", không jank, tôn trọng `prefers-reduced-motion`, và có bản đồ phím tắt thống nhất để engineer implement một lần trong một registry.

## 2. Phạm vi

- Trạng thái hover/focus/active/pressed của mọi control; transition của overlay; loading; toast; progress; real-time update; reduced-motion; keyboard shortcuts toàn cục và theo màn.
- Không bao gồm: animation sprite trong canvas office (chỉ quy định reduced-motion và fps trần).

## 3. Nội dung chi tiết

### 3.1 Nguyên tắc motion

1. Motion chỉ làm 3 việc: định hướng không gian (từ đâu tới), xác nhận (đã xảy ra), báo thay đổi (dữ liệu mới). Nếu không thuộc 3 việc này → không animate.
2. Thời lượng ngắn: 80–280ms; chỉ row-highlight được 1200ms vì là "fade đi" thụ động.
3. "Xuất hiện" dùng `--ease-out`, "biến mất" dùng `--ease-in` và nhanh hơn 30%.
4. Chỉ animate `opacity` và `transform` (GPU); không animate `width/height/top/left` trừ sidebar collapse (dùng `width` + `will-change` có chủ ý) và tab indicator (transform).
5. Không animation vòng lặp ngoài: spinner, status `running` icon spin, pulse `waiting`, edge dash khi truyền dữ liệu, skeleton shimmer.

### 3.2 Trạng thái control

| Trạng thái | Thay đổi | Thời lượng |
|---|---|---|
| hover | nền/border đổi theo variant (`DESIGN_SYSTEM.md` §3.1); icon `text-secondary → text-primary`; cursor pointer chỉ cho link/button | `--duration-instant` 80ms, `background-color, color, border-color` |
| focus-visible | ring 2px `ai-accent` + offset 2px nền `bg`; không đổi nền | 120ms (ring opacity) |
| active/pressed | `transform: scale(0.98)` cho button md/lg; nền tối thêm 1 bậc | 80ms; bỏ scale khi reduced-motion |
| disabled | opacity .5, không hover, `cursor: not-allowed` | không transition |
| loading | spinner thay icon, label giữ, `aria-busy=true`, width khóa | spinner 0.8s linear |
| selected (row/card) | nền `ai-accent-soft` hoặc border `ai-accent` | 120ms |
| drag | `shadow-md`, `rotate(1deg)`, opacity .9; ghost tại chỗ cũ opacity .3 | theo con trỏ, không easing |

Hover trên touch: không áp dụng (`@media (hover: hover)`).

### 3.3 Overlay animations

| Thành phần | Vào | Ra |
|---|---|---|
| Tooltip | opacity 0→1, 120ms, delay 400ms (0ms nếu đã mở tooltip khác trong 300ms) | opacity 80ms |
| Dropdown/Popover/Select | opacity + `scale(0.96→1)` từ origin cạnh trigger, 120ms ease-out | 80ms ease-in |
| Dialog | backdrop opacity 200ms; panel opacity + `scale(0.97→1)` 200ms ease-out | panel 120ms ease-in, backdrop 160ms |
| Sheet/Drawer | `translateX(100%→0)` 280ms ease-out (bottom: translateY); backdrop opacity 200ms | 200ms ease-in |
| Command Palette | opacity + `scale(0.98→1)` 120ms; input autofocus ngay frame đầu | 80ms |
| Toast | `translateY(16px→0)` + opacity 200ms; stack đẩy nhau 200ms ease-standard | opacity + translateX(16px) 160ms |
| Sidebar collapse | `width 240→56` 280ms ease-standard; label fade out 80ms trước khi width đổi | ngược lại, label fade in sau width |
| Route change | nội dung mới opacity 0→1 120ms; không slide; giữ scroll position theo route | không animate ra |
| Tab indicator | `transform: translateX/scaleX` 200ms ease-standard | — |
| Accordion | `grid-template-rows 0fr→1fr` 200ms (CSS-only) | 160ms |

### 3.4 Loading & progress

- **Ngưỡng**: < 200ms không hiện gì; 200ms–5s skeleton/spinner; > 5s progress có ước lượng hoặc nút "Run in background".
- **Skeleton**: xuất hiện sau 200ms delay, hiển thị tối thiểu 400ms, mô phỏng layout thật; chuyển sang nội dung bằng crossfade 120ms (không pop).
- **Progress bar** (task/workflow): `transform: scaleX` với transition 300ms ease-standard giữa các giá trị; không nhảy lùi (nếu server báo lùi, giữ giá trị hiện tại và hiện tooltip "re-estimating"); indeterminate = thanh 30% chạy 1.2s linear vô hạn, tắt thành pulse opacity khi reduced-motion.
- **Inline button loading**: nút giữ width, icon → spinner, disable click, timeout 30s → trả về + toast lỗi.
- **Optimistic update**: hành động nhỏ (đổi status, assign, approve) cập nhật UI ngay, rollback + toast nếu server từ chối; TanStack Query `onMutate` **[Đề xuất]**.

### 3.5 Toast & thông báo

- Chỉ toast cho kết quả hành động của **người dùng** (saved, approved, workflow started, lỗi do người dùng gây ra) và sự kiện cần hành động ngay (agent failed cần retry, approval mới nếu người dùng bật "notify"). Sự kiện tiến độ của agent → Notification center + Activity indicator, không toast.
- Gộp: ≥ 3 toast cùng loại trong 2s → một toast "3 tasks completed" với "View".
- Toast có `role="status"` (info/success) hoặc `role="alert"` (error); pause timer khi hover/focus.

### 3.6 Real-time update patterns (quan trọng nhất) **[Đã xác minh: yêu cầu không jank]**

Nguồn sự kiện: WebSocket/SSE từ orchestrator, có thể tới 18 agent × nhiều event/giây.

1. **Batching**: client gom event trong cửa sổ 100ms (`requestAnimationFrame` + buffer) rồi commit một lần vào Zustand; bảng/list chỉ re-render một lần cho mỗi batch. Log stream: gom 50 dòng hoặc 100ms, append theo chunk.
2. **Row highlight**: hàng vừa thay đổi nhận class `data-updated` → nền `ai-accent-soft` (hoặc `status-*-soft` theo loại thay đổi) fade về trong suốt trong `--duration-highlight` 1200ms, ease-out. Không highlight lại nếu hàng đã highlight trong 1s. Hàng mới chèn: cao 0→40px 200ms + highlight; khi reduced-motion: xuất hiện ngay, vẫn highlight (opacity only).
3. **Giữ vị trí cuộn**: với list đang cuộn (user không ở đầu), item mới không đẩy nội dung; hiện pill "3 new updates ↑" ở đầu list (sticky, `ai-accent`), click để cuộn lên. Log viewer: auto-follow chỉ khi đang ở cuối (tolerance 24px).
4. **Không reorder tức thì**: Kanban/table đang hiển thị không tự di chuyển card khi status đổi nếu người dùng đang hover/kéo; delay 600ms sau khi con trỏ rời. Khi di chuyển: FLIP animation 200ms ease-standard (`transform`), reduced-motion → đổi vị trí ngay + highlight.
5. **Số liệu (metric card)**: số đếm chuyển bằng tween 300ms (`tabular-nums` để không rung chiều ngang); reduced-motion → đổi ngay.
6. **Activity indicator** (topbar): số agent running cập nhật ngay; chấm violet pulse 2s ease-in-out vô hạn chỉ khi ≥ 1 agent running; reduced-motion → chấm tĩnh.
7. **Chart live**: thêm điểm mới không animate; trục trượt bằng transform 300ms khi dữ liệu > viewport.
8. **Workflow canvas**: node đổi trạng thái → border color 200ms; edge đang chạy: `stroke-dasharray 6 4` + `stroke-dashoffset` animate 1s linear vô hạn (reduced-motion: edge màu running tĩnh).
9. **Virtual Office**: nhận cùng batch; sprite chuyển state ở tick tiếp theo (≤ 100ms); cap 30fps render canvas, 12fps animation sprite; khi tab ẩn (`visibilitychange`) dừng render loop.
10. **Back-pressure**: nếu buffer > 500 event chưa xử lý → bỏ event trung gian, chỉ giữ state cuối từng entity (last-write-wins theo `entityId`), hiển thị badge "Live · catching up".

### 3.7 Reduced motion & hiệu năng

- `@media (prefers-reduced-motion: reduce)`: mọi duration → 1ms (token), bỏ `transform` (scale/translate), giữ opacity; tắt shimmer, dash, pulse, spin (spinner → icon tĩnh `loader-circle` + text "Loading"); FLIP → tức thì; office sprite → tĩnh.
- Setting trong app: "Motion: System / Reduced / Full" (`data-motion` trên `<html>`) override media query **[Đề xuất]**.
- Mục tiêu: 60fps khi 18 agent cùng phát event; INP < 200ms; không layout shift khi update (CLS 0 ngoài lúc load đầu).
- `content-visibility: auto` cho card ngoài viewport; `will-change` chỉ trong lúc animate.

### 3.8 Keyboard shortcuts map **[Đề xuất — Cần người dùng phê duyệt]**

Nguồn sự thật cho phím tắt là `NAVIGATION_STRUCTURE.md` §9 (Agent 16); bảng dưới đây **kế thừa nguyên bộ đó** và chỉ bổ sung phím tắt cấp component (form, table, palette). Quy ước: `Mod` = Ctrl (Windows/Linux) / Cmd (macOS). Chord kiểu "G then D" (GitHub/Linear) với timeout 1s. Khi focus trong input/textarea/Monaco/terminal, chỉ `Mod+K` và `Esc` hoạt động ở mức global. Hiển thị trong Tooltip, Command Palette, và màn "Keyboard shortcuts" (`?`).

**Global**

| Phím | Hành động | Nguồn |
|---|---|---|
| `Mod+K` | Command Palette | Agent 16 |
| `/` | Focus global search | Agent 16 |
| `[` | Toggle sidebar | Agent 16 |
| `N` | Notification popover | Agent 16 |
| `?` | Trang shortcuts | Agent 16 |
| `Esc` | Đóng overlay trên cùng / bỏ chọn | Agent 16 |
| `Mod+Shift+.` | Emergency stop all agents (confirm gõ `STOP`) | Agent 16 |
| `Mod+Shift+P` | Toggle theme (dark/light) | bổ sung Agent 11 |
| `\` | Toggle inspector/sheet phải (pin) | bổ sung Agent 11 |

**Điều hướng (chord `G` rồi…)** — Agent 16: `G D` Dashboard · `G O` Office · `G A` Agents · `G T` Tasks · `G P` Approvals · `G W` Workflows · `G C` Code · `G Q` QA · `G S` Settings. Bổ sung: `G H` Provider Hub · `G N` Analytics · `G ,` Org chart.

**List / Table / Form (cấp component — Agent 11)**

| Phím | Hành động |
|---|---|
| `↑ ↓` / `J K` | Di chuyển trong list/table; `Enter` mở; `Space` preview drawer (Agent 16) |
| `Shift+Space` | Toggle chọn hàng; `Mod+A` chọn hết; `Shift+↑↓` chọn dải |
| `Mod+Enter` | Submit form/dialog |
| `Mod+S` | Lưu (settings, workflow, code) |
| `Mod+Z` / `Mod+Y` | Undo/Redo (workflow builder, code) — theo Agent 16 |
| `Mod+1..9` | Chuyển tab trong khu vực hiện tại |
| `Tab` (trong Command Palette) | Sang nhóm sau; prefix `> @ # /` đổi chế độ |

**Theo màn (Agent 16 làm chuẩn; Agent 11 bổ sung in nghiêng)**

| Màn | Phím |
|---|---|
| Task Center | `C` tạo · `A` assign · `1–4` Kanban/Table/Timeline/Graph · *`S` status · `P` priority · `L` label · `D` due · `F` focus filter* |
| Approval Center | `A` mở form Approve · `X` Reject · `M` Request changes · `]` `[` file diff tiếp/trước · *`E` expand/collapse diff* |
| Agent Management | `P` Pause · `R` Resume · `S` Stop (confirm) · *`L` mở log · `T` giao task* |
| Workflow Builder | `Delete` · `Mod+Z/Y` · `Mod+D` duplicate · `Space+drag` pan · `F` fit · `Mod+Enter` run · *`V` select · `+/-` zoom* |
| Code Workspace | Monaco mặc định · `Mod+P` quick open · `` Mod+` `` terminal · `Mod+Shift+G` git · *`Alt+↑/↓` chuyển hunk diff* |
| Virtual Office | `WASD`/mũi tên pan · `+/-` zoom · `F` follow agent · `E` edit mode · `H` HUD · `M` mute · *`Space` tạm dừng animation* |
| Analytics | *`R` đổi khoảng thời gian (7d/30d/90d) · `Mod+E` export CSV* |

Registry: một file `shortcuts.ts` export `{ id, keys, scope, when, handler, label }`; Command Palette và Tooltip đọc từ cùng registry — không hardcode ở component.

### 3.9 Micro-copy cho feedback

- Thành công: "Workflow started" (không "Successfully started the workflow!").
- Lỗi: "Couldn't approve. Agent Nova is offline. Retry" — có động từ hành động cuối.
- Trạng thái chờ: "Waiting for your approval · 4m" hiển thị thời gian chờ để tạo cảm giác khẩn cấp đúng mức.

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Trạng thái |
|---|---|---|---|
| I1 | Batch 100ms cho real-time | Cân bằng độ "live" và số lần render; mắt không phân biệt < 100ms | Đề xuất |
| I2 | Không toast cho progress event | Tránh toast storm với 18 agent | Đã xác minh |
| I3 | Chord `G` + chữ cái cho điều hướng | Chuẩn Linear/GitHub, không chiếm `Mod+` của browser/OS | Đề xuất |
| I4 | Chỉ animate opacity/transform | Tránh layout thrash; cần cho CLS 0 | Đã xác minh (best practice) |
| I5 | Setting Motion trong app override OS | Người dùng desktop hay quên setting OS | Đề xuất |
| I6 | Kéo card: delay reorder tự động 600ms | Tránh card "chạy" dưới tay người dùng | Đề xuất |

## 5. Dependencies

- Agent real-time/orchestrator: hợp đồng event (`entityId`, `version`, `ts`) để last-write-wins hoạt động.
- Agent state (Zustand): batch commit API.
- `DESIGN_TOKENS.md` motion tokens; `DESIGN_SYSTEM.md` component states.
- Thư viện: `motion` (framer-motion) chỉ cho FLIP/layout animations **[Đề xuất]**; còn lại CSS transitions.

## 6. Rủi ro

- Chord shortcuts xung đột IME tiếng Việt (Unikey gõ `G` + chữ) khi focus ngoài input — giảm thiểu: chord chỉ bắt khi `event.isComposing === false`.
- `[` vừa là toggle sidebar (global) vừa là "file diff trước" trong Approval detail (Agent 16) — cần scope: khi focus trong diff viewer, `[`/`]` thuộc diff; ngoài ra thuộc sidebar. Registry phải hỗ trợ `scope` ưu tiên.
- Monaco chiếm nhiều `Mod+` shortcut; trong Code Workspace phải ưu tiên Monaco, app shortcut chỉ khi focus ngoài editor.
- FLIP animation trong table virtualized phức tạp; có thể bỏ FLIP cho table, giữ cho Kanban.

## 7. Vấn đề chưa giải quyết

- Có cho người dùng tùy chỉnh shortcut không? Đề xuất: v2.
- Âm thanh thông báo (approval mới) — mặc định tắt, có bật trong Settings? **[Cần người dùng phê duyệt]**
- Desktop notification (OS-level) cho approval khi app ở background — phụ thuộc Electron/Tauri.

## 8. Acceptance Criteria

- [ ] Mọi transition dùng token duration/easing; không literal `ms` ngoài `tokens.css` (grep).
- [ ] Bật `prefers-reduced-motion` → không có `transform` animation nào chạy; spinner thành tĩnh; office sprite tĩnh.
- [ ] Stress test: 18 agent × 10 event/s trong 60s → FPS trung bình ≥ 55, INP < 200ms, CLS = 0 trên laptop tham chiếu (i5/8GB).
- [ ] Row highlight hiển thị ≤ 100ms sau event, fade đúng 1200ms, không highlight chồng.
- [ ] List đang cuộn không nhảy khi có item mới; pill "new updates" xuất hiện.
- [ ] Tất cả shortcut trong bảng hoạt động, hiển thị trong Tooltip/Palette và trang `?`; không xung đột khi focus trong input.
- [ ] Skeleton không gây layout shift khi thay bằng nội dung thật (CLS đo < 0.01).


> **Phụ lục sửa đổi:** xem `docs/design/HERDR_DESIGN_INTEGRATION.md` (attention state, Agent Rail, token bổ sung). Khi mâu thuẫn, phụ lục thắng cho các hạng mục đó cho đến khi chủ sở hữu duyệt (D-017..D-019).
