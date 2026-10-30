# DESIGN SYSTEM — COMPONENT INVENTORY & RULES

> Phase 0 · Agent 11 · Phiên bản 0.1 · Phụ thuộc `DESIGN_TOKENS.md` (mọi giá trị màu/size tham chiếu token).
> Đánh dấu: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**

## 1. Mục đích

Liệt kê toàn bộ component của AI WORKSPACE, quy định variant/size/state, ánh xạ sang shadcn/ui (Radix) primitive, và chỉ rõ component nào là custom. Engineer dùng tài liệu này để scaffold `src/components/ui/*` (shadcn) và `src/components/app/*` (custom) mà không cần hỏi thêm.

## 2. Phạm vi

- Tất cả component dùng chung ≥ 2 màn hình. Component chỉ dùng trong một màn (ví dụ Org Chart node) được mô tả ở tài liệu màn hình tương ứng, nhưng phải compose từ component ở đây.
- Không bao gồm sprite/UI trong canvas Phaser (Virtual Office).

## 3. Nội dung chi tiết

### 3.0 Quy ước chung

- Thư mục: `components/ui/` = shadcn primitives (không sửa logic, chỉ style qua token); `components/app/` = custom composite.
- Mọi component nhận `className` và forward ref; styling bằng `cva` (class-variance-authority) + `cn()`.
- Size chuẩn: `sm` 28px, `md` 32px (**mặc định**), `lg` 40px cho control; icon trong control: sm 14px, md 16px, lg 16px.
- Focus: `focus-visible:ring-2 ring-ai-accent ring-offset-2 ring-offset-bg`, không `outline`.
- Disabled: `opacity-50 pointer-events-none` + `aria-disabled`.
- Icon luôn `aria-hidden` khi kèm text; icon-only button bắt buộc `aria-label` + Tooltip.

### 3.1 Button — shadcn `Button` **[Đã xác minh (shadcn)]**

| Variant | Nền | Chữ | Border | Hover | Dùng cho |
|---|---|---|---|---|---|
| `primary` | `ai-accent` | `ai-accent-fg` | none | `ai-accent-hover` | 1 hành động chính/màn hoặc dialog |
| `secondary` | `surface` | `text-primary` | `border-strong` | `surface-hover` | hành động phụ |
| `ghost` | transparent | `text-secondary` | none | `surface-hover`, chữ `text-primary` | toolbar, icon button |
| `outline` | transparent | `text-primary` | `border` | `surface-hover` | filter chip, toggle |
| `destructive` | `status-failed` | `#FFF` | none | tối 8% | xóa, dừng agent |
| `link` | none | `link` | none | underline | inline |

Size: `sm` h-28 px-2.5 text-xs · `md` h-32 px-3 text-sm · `lg` h-40 px-4 text-base · `icon` 32×32 (`icon-sm` 28). Radius `sm` (6px). Font-weight 500. State `loading`: thay icon trái bằng `loader-circle` spin, giữ nguyên width (`min-w` theo label), `aria-busy`. Trong dialog: primary bên phải, cancel (`secondary`) bên trái nó; destructive confirm cần type tên hoặc hold 600ms (**[Đề xuất]**).

### 3.2 Input / Textarea / Select / Combobox

- **Input** (shadcn `Input`): h-32, px-3, text-sm, bg `bg-sunken` (dark) / `surface` (light), border `border-strong`, radius sm; hover border `neutral-500`; focus ring accent; error: border `status-failed` + helper text 12px đỏ + `aria-invalid`. Prefix/suffix slot 16px icon màu `text-tertiary`. Mono variant (`font-mono`) cho API key, path, ID.
- **Textarea**: min-h-80, auto-grow tới 240px, cùng style Input; dùng cho prompt/instruction của agent, có counter token ở góc dưới phải (mono xs).
- **Select** (shadcn `Select` — Radix): trigger giống Input, chevron-down 16px; content = Popover style (§3.4), item h-32, check icon bên trái khi chọn.
- **Combobox** (shadcn `Command` trong `Popover`): dùng khi > 8 lựa chọn (chọn agent, chọn skill, chọn provider). Có search, nhóm, item có avatar/icon 16px.
- **Checkbox / Switch / RadioGroup**: shadcn mặc định; Switch 36×20, thumb 16; màu on = `ai-accent`; Checkbox 16×16 radius xs.

### 3.3 Dropdown Menu / Context Menu — shadcn `DropdownMenu`, `ContextMenu`

Content: bg `surface-raised`, border `border`, radius md, shadow-sm, padding 4px, min-w 180 max-w 320. Item: h-32, px-2, text-sm, icon trái 16px `text-secondary`, shortcut hint phải mono xs `text-tertiary`. Separator 1px `border-subtle` margin 4px. Label nhóm: 2xs uppercase `text-tertiary`. Destructive item: chữ `status-failed`. Open animation: fade+scale từ 0.96, 120ms ease-out.

### 3.4 Popover / Tooltip / HoverCard

- **Tooltip** (shadcn `Tooltip`): bg `neutral-100` (light) / `neutral-200` (dark) với chữ ngược (`text-inverse`), text-xs, px-2 py-1, radius xs, không arrow, delay 400ms, không blur. Chứa shortcut: `<kbd>` mono 2xs. Không chứa nội dung tương tác.
- **Popover**: style như Dropdown content, padding 12–16, có thể chứa form nhỏ (ví dụ đổi priority).
- **HoverCard**: dùng cho preview agent/task khi hover ID — delay 600ms, width 320.

### 3.5 Dialog — shadcn `Dialog` / `AlertDialog`

Width: `sm` 400, `md` 520 (**mặc định**), `lg` 720, `xl` 960 (chỉ cho preview diff). Bg `surface-raised`, radius lg, shadow-lg, border `border`. Header: title text-md 500 + description text-sm `text-secondary`; close icon-button góc phải. Body padding 24 (md), cuộn trong body, header/footer cố định. Footer: căn phải, gap 8. Không dialog lồng dialog; multi-step dùng `Stepper` trong body. `AlertDialog` cho destructive, không đóng khi click outside.

### 3.6 Drawer / Sheet — shadcn `Sheet`

Cạnh phải mặc định; width `md` 480, `lg` 640, `xl` 50vw (max 960). Dùng cho: chi tiết task, chi tiết agent, log viewer, approval detail. Có header sticky với breadcrumb nhỏ + action; body cuộn; có thể "pin" thành split panel ở ≥ xl (**[Đề xuất]**). Mobile: cạnh dưới (`bottom`), height 92vh, có handle 36×4 radius full. Nhiều sheet: thay thế nội dung, không chồng (giữ stack trong Zustand để Back).

### 3.7 Tabs — shadcn `Tabs`

Variant `line` (mặc định): trigger h-36, text-sm 500, chữ `text-secondary`, active `text-primary` + indicator 2px `ai-accent` dưới (animate x/width 200ms). Variant `segmented`: nền `surface-hover`, radius sm, pill active `surface` + shadow-sm — dùng cho view switcher (Kanban/Table/Timeline). Tab có badge count: Badge `neutral` size sm bên phải label.

### 3.8 Badge — shadcn `Badge` + custom `StatusBadge`

`Badge` variant: `neutral` (bg `surface-active`, chữ `text-secondary`), `outline`, `ai` (bg `ai-accent-soft`, chữ `ai-accent`). Size: sm h-18 text-2xs px-1.5; md h-20 text-xs px-2. Radius xs (không pill, trừ count).

**StatusBadge** (custom, `components/app/StatusBadge.tsx`): props `status: 'idle'|'running'|'waiting'|'blocked'|'failed'|'completed'|'paused'|'disabled'`, `size`, `showLabel`. Render: dot 6px (hoặc icon 12px) màu `status-*` + label text-xs 500 cùng màu, nền `status-*-soft`, border none, radius xs, h-20. `running` có icon spin; `waiting` có pulse nhẹ (xem `INTERACTION_DESIGN.md`). Luôn có `aria-label="Status: running"`.

### 3.9 Avatar — shadcn `Avatar` + custom `AgentAvatar`

Size 20/24/32/40/64. Radius: người dùng = full; **agent = md (8px)** để phân biệt người/AI **[Đề xuất]**. Fallback: 2 ký tự initials, bg `surface-active`, chữ `text-secondary`.

**AgentAvatar**: Avatar + ring 2px màu agent-state (online green / busy violet / waiting amber / error red / offline none) + **provider mini-badge** 12px (size ≥ 32) hoặc 10px (size 24) ở góc dưới phải: ô vuông radius 3px, bg `surface`, border 1px `border`, bên trong ký tự "C"/"X"/"O" 8px mono 600 màu `provider-*` (hoặc icon SVG provider nếu được cấp phép). Tooltip: "Nova · Claude Code · Busy".

### 3.10 Card — shadcn `Card` (một cấp) **[Đã xác minh]**

Bg `surface`, border 1px `border`, radius md, padding 16 (compact 12), không shadow. Header: title text-md 500 + optional action phải. **Không** đặt Card trong Card; bên trong dùng `Section` (title 2xs uppercase + divider) hoặc list. Card clickable: hover border `border-strong` + bg không đổi; selected: border `ai-accent`. Card metric (Dashboard): value text-xl mono tabular, label text-xs `text-secondary`, delta nhỏ màu status, sparkline 80×24 bên phải.

### 3.11 Data Table — custom trên TanStack Table + shadcn `Table` **[Custom]**

- Row 40 (comfortable) / 32 (compact); header sticky top (`z-sticky`), h-36, text-2xs uppercase `text-tertiary`, bg `surface`, border-bottom `border`.
- Cell text-sm; số mono tabular căn phải; cột ID mono xs `text-secondary`.
- Sort: click header, icon `chevrons-up-down` → `chevron-up/down`; multi-sort Shift+click. Column resize kéo cạnh (min 80). Column visibility qua Dropdown "Columns".
- Row hover `surface-hover`; selected `ai-accent-soft` + checkbox; keyboard ↑↓ di chuyển, Enter mở, Space chọn, Ctrl+A chọn hết.
- Virtualize khi > 100 hàng (`@tanstack/react-virtual`). Empty/Loading/Error state theo §3.17.
- Pagination mặc định: cursor/infinite cho log, page-size 50 cho table; hiển thị "1–50 of 1,204" mono xs.
- Density toggle ở toolbar table, lưu per-table trong localStorage.

### 3.12 Charts — Recharts (hoặc visx) **[Đề xuất: Recharts]**

- Palette `--chart-1..6`; mặc định 1 series = `ai-accent`; so sánh ≤ 3 series.
- **Sparkline**: line 1.5px, không axis, không dot (dot cuối 3px), fill 8% opacity, height 24–32.
- **Line**: stroke 1.5px, `strokeLinejoin: round`, grid ngang `border-subtle` dash 2 2, không grid dọc; axis text-xs mono `text-tertiary`; tooltip = Popover style nhỏ, số mono.
- **Bar**: radius 2px trên, gap 40%, màu đơn; stacked tối đa 4 phần.
- **Donut** chỉ cho tỉ lệ status (dùng màu status), stroke-width 8, không label trong vòng.
- Không 3D, không gradient fill, không animation lúc cập nhật real-time (chỉ animate 300ms lần mount đầu).

### 3.13 Navigation

- **Sidebar** (custom): width 240 mở / 56 thu gọn (icon-only + tooltip), bg `surface`, border-right `border`. Trên cùng: Project Switcher (Combobox, h-40, logo 20 + tên + chevron). Nhóm: label 2xs uppercase `text-tertiary` pt-16 pb-4 (ẩn khi collapsed, thay bằng separator). Item: h-32, px-8, radius sm, icon 20 (`strokeWidth 1.75`), text-sm; active: bg `surface-active`, chữ `text-primary`, thanh 2px `ai-accent` bên trái (**[Đề xuất]**); badge count bên phải (`Badge neutral sm`, đỏ nếu approval chờ). Danh sách group/item/badge/route là của `NAVIGATION_STRUCTURE.md` (Agent 16) — 14 group theo brief, có thể gom 4 cụm Monitor/Build/Organize/Platform bằng divider **[Cần người dùng phê duyệt — theo Agent 16]**; tài liệu này chỉ quy định style. Collapse: phím `[`, nhớ trong localStorage. Cuối sidebar: user menu + theme toggle.
- **Topbar** (custom): h-48, bg `bg`, border-bottom `border`. Trái: breadcrumb (text-sm, separator `chevron-right` 14, item cuối `text-primary`, còn lại `text-secondary`; max 3 cấp, giữa rút gọn "…" dropdown). Giữa: Global search trigger (Input-like, w-320, placeholder "Search or jump to…  /", focus bằng `/`; `⌘K` mở Command Palette). Phải: Agent activity indicator (custom: 3 dot/ số agent running + violet pulse nhỏ, click mở popover danh sách), Provider status (dot xanh/vàng/đỏ + tên provider, popover chi tiết latency/quota), Notifications (bell + badge count), User menu (Avatar 24).
- **Breadcrumb**: shadcn `Breadcrumb`.

### 3.14 Command Palette — shadcn `Command` (cmdk) trong `Dialog` **[Đã xác minh]**

Mở `Ctrl/Cmd+K`; width 640, top 15vh, radius lg, shadow-md, bg `surface-raised`, `z-palette`. Input h-48 text-base không border, icon search trái. Nhóm: "Navigate" (các màn), "Actions" (Create task, Run workflow, Pause all agents…), "Agents", "Tasks" (search theo ID/tên), "Recent". Item h-36: icon 16 + label + mô tả `text-tertiary` + shortcut `<kbd>` phải. Prefix modes: `>` action, `@` agent, `#` task, `/` route (**[Đề xuất]**). Tối đa 8 item/nhóm, "Show more". Animation: fade+scale 0.98→1, 120ms. Footer 28px: hint "↑↓ navigate · ↵ open · esc close".

### 3.15 Toast — shadcn `Sonner`

Vị trí dưới-phải (desktop), trên-giữa (mobile). Width 360, bg `surface-raised`, border, radius md, shadow-md, icon 16 theo loại (info `info`, success `circle-check` green, warning `triangle-alert` amber, error `circle-x` red). Title text-sm 500, mô tả text-xs. Auto-dismiss 5s (error 8s, có action thì 10s); tối đa 3 hiển thị, còn lại xếp. Toast có action "Undo"/"View". Không toast cho sự kiện real-time dồn dập (dùng Notification center + row highlight).

### 3.16 Skeleton — shadcn `Skeleton`

Bg `surface-hover`, radius xs, shimmer `linear-gradient` 1.6s (ngoại lệ gradient duy nhất), tắt khi reduced-motion (static). Skeleton phải mô phỏng đúng layout cuối (số hàng, chiều cao row) để không layout shift. Hiển thị sau 200ms delay (tránh flash) và tối thiểu 400ms khi đã hiện.

### 3.17 Empty / Loading / Error / Success states — custom `StateBlock`

`StateBlock` props `variant: 'empty'|'error'|'success'|'offline'`, `icon`, `title`, `description`, `action?`, `secondaryAction?`. Icon 32 trong ô 48 radius md bg `surface-hover`, title text-md 500, mô tả text-sm `text-secondary` max-w 360, căn giữa, padding 48. Empty luôn có CTA (primary hoặc secondary); error có "Retry" + mã lỗi mono xs + "Copy details". Inline error (trong card) dùng biến thể `compact` h-80. Loading = Skeleton (không spinner toàn trang); spinner chỉ trong nút hoặc góc panel.

### 3.18 Agent status chip — custom `AgentChip`

Pill h-28 (sm 24), bg `surface`, border `border`, radius full, padding-left 2 padding-right 10: AgentAvatar 20 + tên text-sm + StatusBadge dot-only + (optional) hoạt động hiện tại text-xs `text-tertiary` cắt 24 ký tự. Click → HoverCard/Sheet. Dùng trong Task card, Approval card, Topbar activity popover, Workflow node.

### 3.19 Task card — custom `TaskCard` (Kanban) & `TaskRow` (Table)

TaskCard: Card compact padding 12, width theo cột (min 280). Hàng 1: ID mono xs `text-secondary` + priority icon (`chevrons-up` đỏ P0, `chevron-up` cam P1, `minus` xám P2, `chevron-down` xám P3) + menu `ellipsis` ẩn tới hover. Hàng 2: title text-sm 500, 2 dòng clamp. Hàng 3: StatusBadge + labels (Badge outline, max 2 + "+n"). Hàng 4: AgentAvatar 20 (assignee) + due/elapsed mono xs + icon dependency (`git-branch` + count) + approval pending (`shield-alert` amber) nếu có. Drag: cursor grab, khi kéo: shadow-md + rotate 1°; drop target: cột viền dashed `border-strong`. Selected: border `ai-accent`.

### 3.20 Approval card — custom `ApprovalCard`

Card padding 16, viền trái 3px `status-waiting` (duy nhất chỗ dùng viền màu). Header: loại approval (Badge: "Code change" / "Shell command" / "Spend > $X" / "External call"), thời gian chờ mono xs, mức rủi ro (Badge `neutral`/amber/red: Low/Medium/High). Body: tóm tắt 2 dòng + khối preview (diff rút gọn 6 dòng mono / command block / số tiền) bg `bg-sunken` radius sm. Footer: AgentChip người yêu cầu + nút `Reject` (secondary) · `Request changes` (ghost) · `Approve` (primary) · `View details` (ghost). Keyboard (theo `NAVIGATION_STRUCTURE.md` §9): `A` mở form approve, `X` reject, `M` request changes — không áp dụng ngay, luôn qua form xác nhận; High risk thêm bước gõ xác nhận. Mobile: nút full-width stack.

### 3.21 Workflow node (React Flow) — custom `WorkflowNode`

Kích thước 240×auto (min-h 64), bg `surface`, border 1px, radius md, header 32 (icon loại node 16 + tên text-sm 500 + AgentAvatar 20 phải), body text-xs `text-secondary`, handle 8px tròn `border-strong` (hover `ai-accent`). Trạng thái:

| State | Border | Header bg | Chỉ báo |
|---|---|---|---|
| idle | `border` | none | — |
| selected | `ai-accent` 1.5px | none | handles hiện |
| running | `status-running` | `status-running-soft` | progress bar 2px đáy + icon spin |
| completed | `status-completed` | none | `circle-check` xanh ở góc, mờ 70% sau 3s |
| failed | `status-failed` | `status-failed-soft` | `circle-x` + nút "Retry" trong body |
| blocked | `status-blocked` dashed | none | `octagon-alert` + tooltip lý do |
| waiting-approval | `status-waiting` | `status-waiting-soft` | `clock` pulse + nút "Review" |

Edge: stroke 1.5 `border-strong`; đang truyền dữ liệu: `status-running` + dash animate (tắt khi reduced-motion); failed path: `status-failed`. Minimap: bg `bg-sunken`, node màu neutral. Canvas nền: dot grid 16px `border-subtle`.

### 3.22 Các component khác (ánh xạ nhanh)

| Component | Nguồn | Ghi chú |
|---|---|---|
| Accordion, Collapsible | shadcn | Settings, log group |
| Progress | shadcn | h-4 (xs) / h-6; màu `ai-accent`; indeterminate stripe tắt khi reduced-motion |
| Separator | shadcn | `border-subtle` |
| ScrollArea | shadcn | thumb 6px `neutral-600/40` |
| Resizable | shadcn (react-resizable-panels) | Code Workspace, inspector |
| Kbd | custom | mono 2xs, bg `surface-hover`, border, radius xs, h-18 px-1.5 |
| CodeBlock / LogViewer | custom | mono sm, bg `bg-sunken`, line number `text-disabled`, virtualized, auto-follow toggle, ANSI color map sang status colors |
| DiffViewer | custom trên Monaco diff | thêm `rgb(79 194 126 / .12)`, xóa `rgb(240 106 94 / .12)` |
| Stepper | custom | dialog multi-step |
| Timeline (Task) | custom | ô ngày 32px, bar h-24 màu status |
| DependencyGraph | React Flow tái dùng WorkflowNode compact 180×40 |
| NotificationItem | custom | unread dot `ai-accent` 6px |
| ThemeToggle, DensityToggle | custom | Segmented 3 lựa chọn |

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Trạng thái |
|---|---|---|---|
| C1 | shadcn "new-york" style, Tailwind v4 | Mật độ gọn hơn "default", phù hợp 13px body | Đề xuất |
| C2 | Agent avatar radius 8px, user avatar tròn | Phân biệt tức thời AI/người, không cần label | Cần người dùng phê duyệt |
| C3 | Viền trái màu chỉ cho ApprovalCard | Approval là thứ duy nhất "đòi" người dùng hành động | Đề xuất |
| C4 | Recharts thay vì D3 thuần | Tốc độ triển khai; palette tự quản | Đề xuất |
| C5 | Không toast cho real-time events | 18 agent → tránh toast storm | Đã xác minh (brief: không jank) |
| C6 | Size control md = 32px | Chuẩn Linear/GitHub; đủ hit target desktop | Đề xuất |

## 5. Dependencies

- `DESIGN_TOKENS.md` (tất cả màu/size). `INTERACTION_DESIGN.md` (animation của từng component).
- shadcn/ui, Radix, cmdk, sonner, TanStack Table/Virtual, React Flow (`@xyflow/react`), Recharts, Monaco, react-resizable-panels.
- Agent Product: taxonomy priority (P0–P3) và loại approval.

## 6. Rủi ro

- shadcn đặt tên `--accent` trùng khái niệm; đã quy ước `--ai-accent` cho AI (xem Tokens §3.11).
- React Flow node custom cần `memo` chặt — 100+ node với trạng thái real-time dễ re-render.
- Table dense 32px trên touch vi phạm hit target → Compact bị khóa ở tablet/mobile (xem `RESPONSIVE_DESIGN.md`).

## 7. Vấn đề chưa giải quyết

- Logo chính thức của provider có được dùng trong mini-badge hay phải dùng ký tự?
- Có cần "Agent avatar generator" (pixel-art avatar dùng chung office và SaaS)? **[Giả định]**: office sprite và SaaS avatar là hai asset nhưng cùng màu nhận diện.
- Storybook hay Ladle cho catalog component — Agent kiến trúc quyết định.

## 8. Acceptance Criteria

- [ ] Mỗi component trong bảng tồn tại trong Storybook với mọi variant/size/state (kể cả loading/disabled/error).
- [ ] Mọi icon-only button có `aria-label` và Tooltip (lint a11y pass).
- [ ] `StatusBadge` render đúng 8 trạng thái ở cả hai theme với contrast ≥ 3:1 trên soft bg.
- [ ] WorkflowNode có đủ 7 trạng thái thị giác và chuyển trạng thái không layout shift.
- [ ] DataTable: sort/resize/virtualize/keyboard hoạt động với 10k hàng ở 60fps trên laptop tham chiếu.
- [ ] Không có Card lồng Card (review + rule lint `.card .card`).
- [ ] Command Palette mở < 50ms sau phím tắt, không fetch chặn render.


> **Phụ lục sửa đổi:** xem `docs/design/HERDR_DESIGN_INTEGRATION.md` (attention state, Agent Rail, token bổ sung). Khi mâu thuẫn, phụ lục thắng cho các hạng mục đó cho đến khi chủ sở hữu duyệt (D-017..D-019).
