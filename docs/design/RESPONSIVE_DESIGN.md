# RESPONSIVE DESIGN, DENSITY & ACCESSIBILITY

> Phase 0 · Agent 11 · Phiên bản 0.1 · Phụ thuộc `DESIGN_TOKENS.md` §3.9 (breakpoints), `DESIGN_SYSTEM.md` §3.13 (navigation).
> Đánh dấu: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**

## 1. Mục đích

Quy định app shell và từng màn hình hành xử thế nào ở desktop/laptop/tablet/mobile, màn nào được hỗ trợ trên mobile (monitoring, approvals) và ở mức nào, chế độ mật độ (density), checklist WCAG 2.2 AA và checklist kiểm thử — để engineer và QA có tiêu chí đo được.

## 2. Phạm vi

- Web build và desktop build (cửa sổ desktop có thể resize nhỏ tới 1024px — hành xử như laptop/tablet).
- Mobile = "basic monitoring & approval" **[Đã xác minh]**, không phải full feature.
- Không bao gồm: native mobile app, in ấn (print stylesheet chỉ cho QA report — tối thiểu).

## 3. Nội dung chi tiết

### 3.1 Breakpoints và thiết bị mục tiêu

| Tên | Range | Thiết bị tham chiếu | Mức hỗ trợ |
|---|---|---|---|
| `xs` | < 640 | Phone dọc (375–430) | Monitoring + Approvals + Notifications + Settings cơ bản |
| `sm` | 640–767 | Phone ngang, phone lớn | như xs, 2 cột ở Dashboard |
| `md` | 768–1023 | iPad dọc, Surface nhỏ | Hầu hết màn ở chế độ "read + act nhẹ"; không Workflow Builder, không Code Workspace edit |
| `lg` | 1024–1279 | Laptop 13" (scale 125%), iPad ngang | Full feature; sidebar tự thu gọn 56px |
| `xl` | 1280–1535 | **Laptop chuẩn — thiết kế chính** (1366/1440/1536 @100–125%) | Full feature, sidebar mở 240 |
| `2xl` | ≥ 1536 | Desktop 1920+, màn ngoài | Full; cho phép split view (sheet pin thành panel), max content width 1600 cho prose/settings, bảng/canvas full-bleed |

Mobile-first trong CSS (`min-width` media queries, Tailwind `md:` `lg:`), nhưng **thiết kế visual-first ở xl** rồi thu về.

### 3.2 App shell theo breakpoint

| Thành phần | ≥ xl | lg | md | ≤ sm |
|---|---|---|---|---|
| Sidebar | 240 mở, có thể collapse 56 (nhớ) | mặc định 56 icon-only + tooltip; mở rộng thành overlay 240 khi hover 300ms hoặc phím `[` | ẩn; mở overlay 280 từ trái bằng nút hamburger + backdrop | ẩn; thay bằng **bottom tab bar** 56px theo `NAVIGATION_STRUCTURE.md` §12: Home · Agents · Approvals (badge) · Activity · More |
| Topbar | 48: breadcrumb · search 320 · activity · provider · bell · user | như xl, search thu 240 | 48: hamburger · title (thay breadcrumb) · search icon · bell · user | 48: title · activity dot · bell; search qua tab "More"/palette |
| Command Palette | 640 | 640 | 90vw | full-screen sheet (input trên cùng, keyboard ảo) |
| Inspector/Sheet phải | 480 (pin được) | 480 overlay | 100vw overlay (có back) | bottom sheet 92vh |
| Page padding | 24 | 24 | 16 | 16 |
| Content max-width | không (grid/table), 1600 cho prose | — | — | — |
| Dialog | md 520 | 520 | min(520, 90vw) | full-screen sheet trừ AlertDialog (90vw center) |
| Toast | dưới-phải | dưới-phải | trên-giữa | trên-giữa, full-width − 32 |

Sidebar overlay ở md/sm bẫy focus (focus trap) và đóng bằng `Esc`/backdrop.

### 3.3 Hành vi từng màn hình

| Màn | ≥ lg | md (tablet) | ≤ sm (mobile) |
|---|---|---|---|
| Dashboard | grid 12 cột: 4 metric card (3 col mỗi) · hàng 2: Active agents (8) + Pending approvals (4) · hàng 3: Activity feed (6) + Cost chart (6) | 2 cột metric, các block stack full | 1 cột; metric card thành 2×2 nhỏ; feed là list |
| Virtual Office | canvas fit viewport, inspector phải | canvas fit, inspector bottom sheet; pan bằng touch | **chỉ xem** canvas thu nhỏ (zoom to fit, không tương tác chi tiết) hoặc danh sách agent-state thay thế **[Đề xuất: danh sách]** |
| Agent Management | table/grid + sheet detail | grid card 2 cột, sheet full | list card 1 cột: avatar, tên, status, hoạt động, nút Pause/Resume |
| Org Chart | React Flow canvas | canvas pan/zoom, chỉ đọc | list phân cấp (tree) dạng accordion |
| Skills | table + detail sheet | table ẩn cột phụ | list; enable/disable toggle |
| Workflow Builder | canvas + palette trái 240 + inspector phải 320 | **chỉ xem** canvas + run/stop; không chỉnh sửa (banner "Edit on desktop") | không hỗ trợ; hiện trạng thái run dạng list step |
| Task Center | Kanban/Table/Timeline/Graph | Kanban cuộn ngang (cột 280), Table ẩn cột phụ, Timeline cuộn ngang, Graph chỉ xem | chỉ **List** (Table rút gọn: title, status, agent); filter trong sheet; đổi status qua action sheet |
| Approval Center | list trái 400 + detail phải | list, detail sheet full | **ưu tiên cao nhất**: list card; detail full-screen với diff cuộn ngang; nút Approve/Reject sticky đáy 56px, full-width |
| Code Workspace | Monaco + explorer + terminal (resizable) | chỉ đọc file + diff (Monaco readOnly) | chỉ xem diff rút gọn (file list + chunk), không editor |
| QA & Security | report table + detail | table ẩn cột, detail sheet | list finding: severity badge + title; detail sheet |
| Provider Hub | card grid 3 cột + detail | 2 cột | 1 cột; trạng thái + quota; không đổi API key (an toàn) **[Đề xuất]** |
| Analytics | chart grid 2 cột | 1 cột, chart height 240 | 1 cột, chart height 180, chỉ 7d/30d, có sparkline |
| Settings | 2 cột: nav trái 220 + form 640 | nav trên (tabs cuộn ngang) + form | tabs cuộn ngang + form; chỉ: Theme, Motion, Notifications, Account |

Chuyển hướng tự động: route không hỗ trợ ở breakpoint hiện tại → hiện `StateBlock` "Available on desktop" + nút quay về Dashboard, không redirect im lặng.

### 3.4 Touch & input

- Hit target ≥ 44×44 ở md trở xuống (padding tăng, không đổi icon size); desktop ≥ 32×32 **[Đã xác minh: không tiny hit target]**. Khoảng cách giữa target ≥ 8px.
- Hover-only tương tác (menu `ellipsis` hiện khi hover) phải luôn hiện ở `(hover: none)`.
- Kéo thả Kanban trên touch: long-press 250ms rồi kéo; cung cấp thay thế "Move to…" trong menu.
- Pinch-zoom cho canvas (office, workflow, org chart); không chặn zoom trình duyệt ở trang thường (`user-scalable` không bị tắt).
- Safe area: `env(safe-area-inset-*)` cho bottom tab bar và toast.

### 3.5 Density modes **[Đề xuất — Cần người dùng phê duyệt về mặc định]**

| Mode | Row | Control | Padding card | Font body | Áp dụng |
|---|---|---|---|---|---|
| Comfortable (**mặc định**) | 40px | 32px | 16px | 13px | toàn app |
| Compact | 32px | 28px | 12px | 13px (12 trong table) | Table, Log, Kanban card; khóa ở `< lg` |
| Spacious | 48px | 36px | 20px | 14px | tùy chọn accessibility; tự bật ở `(pointer: coarse)` |

Cơ chế: `data-density` trên `<html>` + CSS var `--row-h`, `--control-h`, `--card-p`; component đọc var, không nhân class. Lưu trong Settings (global) và override per-table.

### 3.6 Theme & hệ thống

- `data-theme="dark|light"`, "System" lắng nghe `matchMedia('(prefers-color-scheme)')`; chuyển theme không transition màu toàn trang (tránh flash — thêm class `no-transition` 1 frame).
- `color-scheme` set đúng để scrollbar/form control native khớp theme.
- Zoom trình duyệt 200% và Windows scale 150% phải không vỡ layout (dùng rem cho text/control, px cho border/radius).
- `prefers-contrast: more` → border `border-strong`, text-tertiary → text-secondary **[Đề xuất]**.

### 3.7 Accessibility checklist (WCAG 2.2 AA — các mục liên quan)

**Perceivable**
- [ ] 1.1.1 Mọi icon có ý nghĩa có text thay thế; icon trang trí `aria-hidden`.
- [ ] 1.3.1 Table dùng `<th scope>`; form có `<label for>`; nhóm dùng `fieldset/legend` hoặc `role=group` + `aria-labelledby`.
- [ ] 1.3.2 Thứ tự DOM = thứ tự thị giác (không dùng `order` CSS để đảo nội dung quan trọng).
- [ ] 1.4.1 Status không chỉ bằng màu: luôn kèm icon hoặc label (StatusBadge).
- [ ] 1.4.3 Contrast text ≥ 4.5:1; text lớn (≥ 24px hoặc 19px bold) ≥ 3:1 — đo ở cả hai theme.
- [ ] 1.4.4 Zoom 200% không mất nội dung/chức năng.
- [ ] 1.4.10 Reflow ở 320px CSS không cuộn 2 chiều (ngoại lệ: table, canvas, diff).
- [ ] 1.4.11 Border input, focus ring, icon button ≥ 3:1 với nền.
- [ ] 1.4.12 Text spacing override không vỡ layout.
- [ ] 1.4.13 Tooltip/HoverCard đóng bằng `Esc`, không che trigger, hover được vào nội dung.

**Operable**
- [ ] 2.1.1 / 2.1.2 Tất cả thao tác bằng bàn phím; không focus trap ngoài dialog/sheet (và dialog có Esc).
- [ ] 2.1.4 Shortcut ký tự đơn (`C`, `A`, `R`…) chỉ hoạt động khi focus ngoài input và có thể tắt trong Settings.
- [ ] 2.2.1 Toast auto-dismiss có thể tắt/kéo dài (Settings: "Toast duration") ; approval không hết hạn do UI.
- [ ] 2.2.2 Mọi animation lặp > 5s (sprite office, pulse) có nút tạm dừng hoặc tuân theo reduced-motion.
- [ ] 2.3.1 Không flash > 3 lần/giây (row highlight 1 lần/1.2s; cap).
- [ ] 2.4.1 Skip link "Skip to content" đầu DOM; landmark `nav/main/aside/header`.
- [ ] 2.4.3 Thứ tự focus hợp lý; sau khi đóng dialog, focus về trigger.
- [ ] 2.4.7 Focus visible: ring 2px accent + offset, tỉ lệ ≥ 3:1.
- [ ] 2.4.11 (2.2) Focus not obscured: sticky header/footer không che phần tử focus (scroll-margin-top 56px).
- [ ] 2.5.3 Label hiển thị = accessible name (không `aria-label` khác label).
- [ ] 2.5.7 (2.2) Dragging: mọi kéo-thả (Kanban, node, resize) có thay thế không kéo (menu "Move to", input số).
- [ ] 2.5.8 (2.2) Target size ≥ 24×24 CSS px tối thiểu — hệ thống đặt 32/44.

**Understandable**
- [ ] 3.1.1 `lang` trên `<html>` theo locale.
- [ ] 3.2.1 / 3.2.2 Focus/thay đổi input không tự submit hoặc điều hướng.
- [ ] 3.2.6 (2.2) Help nhất quán: nút "?" shortcuts và link Docs ở cùng chỗ (user menu) mọi màn.
- [ ] 3.3.1 / 3.3.3 Lỗi form chỉ rõ trường + cách sửa; `aria-describedby` tới helper.
- [ ] 3.3.7 (2.2) Không bắt nhập lại thông tin đã nhập trong cùng flow (wizard tạo agent).
- [ ] 3.3.8 (2.2) Đăng nhập không yêu cầu nhớ/ghi lại (hỗ trợ paste, password manager).

**Robust**
- [ ] 4.1.2 Dùng Radix primitives (có ARIA); custom component (DataTable, WorkflowNode, AgentChip) khai báo `role`, `aria-selected`, `aria-expanded`.
- [ ] 4.1.3 Status message: toast `role=status/alert`; live region `aria-live=polite` cho "3 new updates", `assertive` chỉ cho failed/approval cần hành động.
- [ ] Canvas (office, workflow): cung cấp bản DOM tương đương (danh sách agent/node) qua `aria-hidden` canvas + visually-hidden list hoặc chế độ "List view".

### 3.8 Testing checklist

**Viewport matrix** (Playwright projects): 375×812 (iPhone), 390×844, 768×1024 (iPad dọc), 1024×768, 1280×800, 1366×768 @125% DPR 1.25, 1440×900, 1920×1080. Mỗi màn chụp snapshot 2 theme × 3 density (chỉ lg+ cho Compact).

**Chức năng**
- [ ] Tab qua toàn bộ màn không mất focus; `Esc` thoát mọi overlay; focus trở về trigger.
- [ ] Chạy axe-core trên từng route (0 violation serious/critical) ở cả hai theme.
- [ ] Contrast script tự động trên token (Tokens §8).
- [ ] Reduced-motion: Playwright `reducedMotion: 'reduce'` → không có `transform` animation (kiểm tra `getAnimations()` rỗng ngoài opacity).
- [ ] Zoom 200% ở 1280 → không cuộn ngang trừ table/canvas.
- [ ] Touch: Kanban long-press kéo và menu "Move to" đều hoạt động trên iPad.
- [ ] Mobile Approval: approve/reject từ 375px trong ≤ 3 chạm từ thông báo.
- [ ] Screen reader smoke (NVDA trên Windows, VoiceOver iOS): Dashboard, Approval detail, Task list đọc được status và hành động.
- [ ] Hiệu năng: Lighthouse perf ≥ 90 (web), INP < 200ms, LCP < 2.5s trên 1366×768 laptop tham chiếu; real-time stress theo `INTERACTION_DESIGN.md` §8.
- [ ] Windows scale 150% + dark theme: border 1px không biến mất (dùng `0.5px`? — không; giữ 1px, kiểm tra).

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Trạng thái |
|---|---|---|---|
| R1 | Thiết kế chính ở xl (1280–1535) | Laptop Windows phổ biến 1366/1536@125% | Đã xác minh (brief: laptop/desktop first) |
| R2 | Mobile = bottom tab bar 5 mục (Home/Agents/Approvals/Activity/More — thống nhất với Agent 16) | Monitoring/approval cần truy cập 1 chạm; sidebar hamburger chậm hơn | Đề xuất |
| R3 | Workflow Builder & Code edit không hỗ trợ < lg | Canvas/Monaco không khả dụng trên touch nhỏ; tránh nửa vời | Đề xuất |
| R4 | Compact khóa ở < lg | Row 32px vi phạm hit target touch | Đã xác minh (WCAG 2.5.8) |
| R5 | Office trên mobile = danh sách agent-state | Pixel canvas trên 375px không đọc được; mục tiêu mobile là monitoring | Cần người dùng phê duyệt |
| R6 | Không đổi API key trên mobile | Giảm rủi ro bảo mật trên thiết bị dùng chung | Đề xuất |

## 5. Dependencies

- Agent kiến trúc: router có "capability per breakpoint" (route meta `minBreakpoint`).
- Agent Security: xác nhận R6 và chính sách approval trên mobile (có cần re-auth/biometric?).
- Agent QA: tích hợp axe + Playwright matrix vào CI.
- `DESIGN_TOKENS.md` (breakpoints, density vars), `DESIGN_SYSTEM.md` (component responsive props).

## 6. Rủi ro

- Desktop app resize nhỏ (< 1024) ít gặp nhưng phải không vỡ → dùng cùng logic breakpoint.
- Bottom sheet trên iOS Safari với keyboard ảo có bug viewport (`100dvh`); cần test thật.
- Hai bản DOM cho canvas (a11y list) có thể lệch dữ liệu với canvas — phải render từ cùng store.
- Compact density + Inter 12px trong table tiệm cận giới hạn đọc; không cho dưới 12px.

## 7. Vấn đề chưa giải quyết

- Mobile có cần PWA (install, push notification cho approval) không? **[Cần người dùng phê duyệt]**
- Approval trên mobile có yêu cầu xác thực lại (PIN/biometric) với High risk?
- Có hỗ trợ RTL không? Giả định: không ở v1.
- Tablet landscape 1024 có nên tính là lg (full feature) hay md? Hiện tại: lg theo width, nhưng `(pointer: coarse)` vẫn ép hit target 44.

## 8. Acceptance Criteria

- [ ] Mọi route khai báo `minBreakpoint`; route không hỗ trợ hiển thị StateBlock "Available on desktop" thay vì lỗi.
- [ ] Snapshot matrix 8 viewport × 2 theme pass, không cuộn ngang ngoài table/canvas/diff.
- [ ] Hit target: audit tự động (axe `target-size`) 0 vi phạm ở md trở xuống (44) và lg trở lên (32 — custom rule).
- [ ] axe-core 0 serious/critical trên 13 màn × 2 theme.
- [ ] Density switch đổi `--row-h/--control-h/--card-p` không reload; Compact bị disable ở < lg với tooltip giải thích.
- [ ] Flow "nhận thông báo → mở approval → approve" hoàn thành trên 375px ≤ 3 chạm, nút sticky không bị keyboard che.
- [ ] Skip link, landmarks, focus return, live region kiểm tra bằng NVDA pass theo script QA.
