# UI/UX GUIDELINES — AI WORKSPACE

> Phase 0 · Agent 11 (UI/UX Designer — Design System Lead) · Phiên bản 0.1 · Trạng thái: Draft chờ owner phê duyệt
> Quy ước đánh dấu: **[Đã xác minh]** (xuất phát từ brief của owner) · **[Đề xuất]** (quyết định của Design Lead, có lý do) · **[Giả định]** (chưa có dữ liệu, cần xác nhận) · **[Cần người dùng phê duyệt]** (owner phải chốt trước Phase 1)

## 1. Mục đích

Tài liệu này là "hiến pháp" thị giác và trải nghiệm của AI WORKSPACE. Mọi tài liệu design khác (`DESIGN_TOKENS.md`, `DESIGN_SYSTEM.md`, `INTERACTION_DESIGN.md`, `RESPONSIVE_DESIGN.md`) đều phải tuân theo các nguyên tắc ở đây. Mục tiêu: một frontend engineer đọc xong bộ 5 tài liệu có thể implement token và component mà không cần hỏi thêm quyết định design.

## 2. Phạm vi

- Áp dụng cho toàn bộ app shell và 13 khu vực chức năng: Dashboard, Virtual Office, Agent Management, Org Chart, Skills, Workflow Builder, Task Center, Approval Center, Code Workspace, QA & Security, Provider Hub, Analytics, Settings.
- Áp dụng cho desktop (Electron/Tauri — do Agent kiến trúc quyết định) và web build dùng chung một codebase React.
- Không bao gồm: marketing site, brand bên ngoài sản phẩm, pixel-art asset pipeline chi tiết cho Virtual Office (chỉ quy định ranh giới cùng tồn tại — mục 3.6).

## 3. Nội dung chi tiết

### 3.1 Triết lý thiết kế — "Calm Control Room"

AI WORKSPACE là nơi một người điều hành tới 18 AI employee chạy song song. Người dùng không "chơi" với app; họ **giám sát, quyết định và can thiệp**. Vì vậy giao diện phải giống phòng điều khiển yên tĩnh: nhiều thông tin, nhưng thứ bậc rõ ràng, màu sắc tiết chế, chỉ nhấn mạnh khi thật sự có việc cần người dùng hành động (approval, failed, blocked).

Năm giá trị cốt lõi từ owner **[Đã xác minh]** và cách diễn giải thành quy tắc:

| Giá trị | Diễn giải thành quy tắc thiết kế |
|---|---|
| Premium | Bề mặt phẳng, border 1px mảnh, shadow gần như không thấy, khoảng trắng chính xác theo grid 4px, typography chặt chẽ (letter-spacing âm nhẹ ở heading). Không có yếu tố trang trí không mang thông tin. |
| Professional | Ngôn ngữ UI trung tính, ngắn gọn, ưu tiên động từ + danh từ ("Approve", "Retry task"). Không emoji trong UI chính. Mật độ thông tin cao như Linear/GitHub nhưng có nhịp thở. |
| Intelligent | Yếu tố do AI tạo ra hoặc đang thực thi được đánh dấu bằng **một** màu accent AI (violet) duy nhất. Hệ thống luôn giải thích trạng thái: "Waiting for your approval", "Blocked by TASK-123". |
| User-friendly | Mọi hành động chính có keyboard shortcut; Command Palette là lối vào thứ hai cho mọi chức năng. Empty state luôn có hành động tiếp theo. Lỗi luôn kèm cách khắc phục. |
| Consistent | Một bộ token, một bộ component, một bộ icon (lucide), một kiểu motion. Không có "ngoại lệ vì màn này đặc biệt" — trừ Virtual Office được quy định riêng ở mục 3.6. |

### 3.2 Nguyên tắc (Principles)

1. **Status trước, trang trí sau.** Màu chỉ xuất hiện khi mang nghĩa (status, AI, destructive, link). Phần còn lại là thang xám trung tính.
2. **Một cấp độ bề mặt.** Page background → Surface (card/panel) → hết. Không card lồng trong card. Phân nhóm bên trong card dùng divider hoặc khoảng cách, không dùng viền thứ hai.
3. **Mật độ có kiểm soát.** Mặc định density "Comfortable" (row 40px); bảng dữ liệu và log cho phép "Compact" (row 32px). Xem `RESPONSIVE_DESIGN.md` §3.5.
4. **Chuyển động có mục đích.** Motion chỉ để (a) giữ ngữ cảnh không gian (drawer trượt từ cạnh), (b) xác nhận hành động (toast), (c) báo thay đổi dữ liệu real-time (row highlight). Không animation trang trí, không loop vô hạn ngoài spinner/indicator "running".
5. **Keyboard là công dân hạng nhất.** Mọi thứ click được phải Tab tới được và có focus ring hiển thị rõ.
6. **Text đọc được.** Body tối thiểu 13px; 11px chỉ dùng cho label/metadata in hoa với tracking rộng; tương phản ≥ 4.5:1 cho text, ≥ 3:1 cho UI element.
7. **Monospace cho máy, sans cho người.** ID, hash, log, path, token count, JSON → mono. Mọi thứ khác → sans.
8. **Không để người dùng chờ mà không biết.** Mọi thao tác > 300ms có feedback (skeleton, progress, spinner inline trên nút). Mọi thao tác > 5s có thể hủy hoặc chạy nền.

### 3.3 Do / Don't

**Do**
- Dùng `border` 1px màu `--color-border` để phân tách vùng; shadow chỉ cho lớp nổi (popover, dialog, drawer).
- Dùng một màu accent AI (violet) cho: avatar ring của agent đang chạy, badge "AI suggested", node đang thực thi, cursor của AI trong Code Workspace.
- Dùng icon lucide ở 16px (inline/table/button) và 20px (sidebar/nav), `strokeWidth=1.75` — thống nhất toàn app.
- Căn chữ số theo `tabular-nums` trong bảng, metric, timer.
- Viết label ngắn, sentence case ("Create workflow", không "Create Workflow" — trừ section label 11px uppercase).
- Hiển thị thời gian tương đối ("2 min ago") kèm tooltip thời gian tuyệt đối ISO.
- Cho mọi bảng/list: sort, filter, keyboard navigation (↑↓ chọn, Enter mở, Space chọn nhiều).

**Don't**
- Không gradient nền, không neon, không glow (trừ pulse rất nhẹ ở status "running" — xem `INTERACTION_DESIGN.md`).
- Không glassmorphism (`backdrop-blur`) trên panel nội dung; chỉ cho phép `backdrop-blur-[2px]` ở overlay dialog.
- Không card trong card; không radius > 12px cho container (trừ avatar/pill).
- Không quá 1 màu accent + status colors trong cùng một màn hình.
- Không icon từ nhiều bộ khác nhau; không icon màu rực (icon kế thừa `currentColor`).
- Không popup chồng popup (dialog mở dialog) — dùng step trong cùng dialog hoặc drawer.
- Không text < 11px; không hit target < 32×32px (desktop) / 44×44px (touch).
- Không auto-play animation nền; không parallax.
- Không dùng màu đỏ cho bất cứ thứ gì không phải lỗi/destructive.

### 3.4 Định hướng thị giác (Visual direction)

- **Nền tối là mặc định** (`#0B0C0F` page, `#131519` surface) — theo brief "neutral dark/charcoal" **[Đã xác minh]**. Light theme là bản đối xứng, không phải "đảo màu": page `#FAFAFA`, surface `#FFFFFF`, border `#E4E4E7`.
- **Màu**: thang xám có sắc lạnh rất nhẹ (zinc-like, hue ≈ 260 trong oklch, chroma 0.004–0.012) để không bị "bẩn" khi đặt cạnh violet. Accent AI: violet `oklch(0.62 0.19 285)` ≈ `#7C6CF5` **[Đề xuất — Cần người dùng phê duyệt]** (phương án thay thế: blue `#3B82F6`). Lý do chọn violet: tách bạch với status "running" (blue) và link (blue) — nếu accent cũng blue, "AI đang làm" và "đang chạy" sẽ trộn nghĩa.
- **Typography**: Inter (UI) + JetBrains Mono (code) **[Đề xuất — Cần người dùng phê duyệt]**; phương án thay thế Geist + Geist Mono. Xem `DESIGN_TOKENS.md` §3.3.
- **Hình khối**: radius 6px cho control, 8px cho card/panel, 12px cho dialog/drawer/palette. Border 1px. Divider 1px cùng màu border.
- **Chiều sâu**: tối đa 3 lớp: base (page), raised (card — không shadow, chỉ border), overlay (popover/dialog — shadow mềm). Light theme dùng shadow nhẹ; dark theme chủ yếu dựa vào chênh lệch độ sáng bề mặt.
- **Iconography**: `lucide-react`, stroke 1.75, size 16/20, màu `--color-text-secondary` ở trạng thái nghỉ, `--color-text-primary` khi hover/active.
- **Data viz**: 1 màu chủ (accent-ai hoặc neutral) + status colors; không rainbow. Grid line `--color-border-subtle`; không fill area dưới line chart (ngoại lệ: sparkline có fill 8% opacity).

### 3.5 Moodboard (mô tả bằng chữ)

Hình dung một bảng 6 ô:

1. **Linear — Issue list**: hàng 36–40px, chữ 13px, status icon nhỏ bên trái, metadata mono mờ bên phải. → Chuẩn cho Task Center table view.
2. **Vercel — Deployment detail**: nền gần đen, panel chỉ khác nền 3–4% độ sáng, log mono cuộn, badge status xám/xanh/đỏ nhỏ gọn. → Chuẩn cho Agent detail / log stream.
3. **Raycast — Command bar**: palette nổi giữa màn, radius 12px, shadow lớn nhưng mềm, nhóm kết quả có section label uppercase 11px. → Chuẩn cho Command Palette.
4. **Stripe Dashboard — Charts**: line chart 1.5px, một màu, không fill, tooltip nhỏ có số mono. → Chuẩn cho Analytics.
5. **GitHub — PR review**: diff mono 13px, comment thread gọn, nút primary nhỏ. → Chuẩn cho Code Workspace & QA report.
6. **Figma — Canvas + panels**: canvas trung tính, panel phải 280–320px, node viền 1px chỉ sáng lên khi chọn. → Chuẩn cho Workflow Builder & Org Chart.

Cảm xúc tổng thể: "tĩnh, chính xác, đắt tiền nhờ tiết chế" — không phải "đắt tiền nhờ hiệu ứng".

### 3.6 Virtual Office và khu SaaS cùng tồn tại

Virtual Office (2D pixel-art, Phaser/Pixi) là khu vực duy nhất được phép có phong cách "game" **[Đã xác minh]**. Quy tắc ranh giới:

- **Canvas pixel-art chỉ nằm trong viewport nội dung** của route `/office`. App shell (sidebar, topbar, command palette, toast, dialog) vẫn là SaaS style 100% — không pixel font, không pixel border ở shell.
- **Palette của office được "dịch" từ token SaaS**: nền sàn/tường dùng neutral-800/900 của dark theme; màu trạng thái nhân vật (bong bóng "coding", "waiting approval") dùng đúng status colors của hệ thống (có thể lượng tử hóa cho pixel-art nhưng hue không đổi). Ở light theme, office đổi bộ tile sáng (tilemap variant) — **[Giả định]** cần Agent Virtual Office xác nhận chi phí asset.
- **Overlay UI trên canvas** (tên agent, tooltip khi hover nhân vật, panel chi tiết khi click) là React/shadcn component thường, render ở lớp DOM phía trên canvas, không vẽ bằng Phaser. Giữ accessibility (screen reader, keyboard) và đồng nhất thị giác.
- **Pixel font** ("Silkscreen"/"Press Start 2P") chỉ dùng cho sprite label *bên trong* canvas, scale nguyên (1x/2x/3x). Mọi text ngoài canvas dùng Inter.
- **Motion**: nhân vật pixel di chuyển 8–12 fps có chủ ý (retro feel) nhưng tôn trọng `prefers-reduced-motion`: dừng đi lại, chỉ đổi sprite frame khi đổi status.
- **Chuyển cảnh** từ office sang màn khác: cùng fade 120ms như mọi route.
- **Khung bao canvas**: border 1px `--color-border`, radius 8px, nền `--color-surface`; canvas không tràn viền và không có "khung gỗ"/trang trí game.

### 3.7 Nguyên tắc ngôn ngữ (UX writing)

- Ngôn ngữ UI mặc định: tiếng Anh (i18n-ready, key-based) **[Giả định — Cần người dùng phê duyệt]**; tiếng Việt là locale thứ hai.
- Sentence case; không dấu chấm cuối label; thông báo lỗi = [Chuyện gì] + [Vì sao nếu biết] + [Làm gì tiếp].
- Tránh "Oops", "Uh-oh"; hệ thống không xưng "I"; agent được gọi bằng tên agent.
- Số liệu: `1,204` (dấu phẩy nghìn), `12.5k` khi ≥ 10k ở chỗ chật, thời gian `2m 13s`, token `1.2M`.

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Trạng thái |
|---|---|---|---|
| D1 | Dark theme mặc định, Light ngang hàng (không phải phụ) | Brief yêu cầu Dark/Light/System; ops console thường dùng dark; light cần cho tablet/mobile ban ngày | Đã xác minh |
| D2 | Accent AI = violet, không trùng blue "running" | Tránh trộn nghĩa "AI" vs "đang chạy" | Cần người dùng phê duyệt |
| D3 | Inter + JetBrains Mono | Phổ biến, hinting tốt trên Windows ClearType, có tabular-nums, OFL | Cần người dùng phê duyệt |
| D4 | Một cấp card | Brief cấm card lồng card; giảm noise | Đã xác minh |
| D5 | Office UI overlay bằng DOM, không vẽ trong canvas | Accessibility + đồng nhất | Đề xuất |
| D6 | lucide strokeWidth 1.75 (thay vì mặc định 2) | Nhẹ hơn, hợp text 13px; vẫn đủ nét ở 16px | Đề xuất |
| D7 | Ngôn ngữ UI mặc định tiếng Anh | Thuật ngữ kỹ thuật, tránh dịch nửa vời | Cần người dùng phê duyệt |

## 5. Dependencies

- Agent kiến trúc frontend (app shell routing, ThemeProvider, Zustand store cho theme/density).
- Agent Virtual Office (asset pipeline, tilemap theo theme).
- Agent Product (danh sách trạng thái task/agent chính thức — 8 trạng thái ở `DESIGN_TOKENS.md` là đề xuất tổng hợp).
- Agent QA/Accessibility (kiểm thử contrast, keyboard).
- shadcn/ui bản hỗ trợ Tailwind v4 (CSS variables theo `@theme`).

## 6. Rủi ro

- Violet accent có thể bị cảm nhận "quá AI-trend"; giảm thiểu bằng chroma vừa phải (0.19) và dùng rất tiết chế.
- Light theme với pixel-art office có thể cần bộ tile riêng → tăng chi phí asset.
- Inter 13px trên Windows scale 100% có thể hơi mảnh; cần test ClearType; fallback: tăng weight body lên 450 với variable font.
- 18 agent viết song song → nguy cơ lệch token nếu không có một nguồn sự thật: `DESIGN_TOKENS.md` là nguồn duy nhất.

## 7. Vấn đề chưa giải quyết

- Ngôn ngữ UI mặc định (EN hay VI).
- Có theme "high contrast" riêng hay chỉ dựa vào `prefers-contrast`?
- Có cho user đổi accent color (như Linear) không? Đề xuất: không ở v1.
- Logo/wordmark AI WORKSPACE chưa có — placeholder là monogram "A" trong ô 24px radius 6px.

## 8. Acceptance Criteria

- [ ] Mọi màn hình (trừ canvas office) chỉ dùng màu từ `DESIGN_TOKENS.md`; lint CSS cấm hex literal ngoài file token.
- [ ] Không có `backdrop-filter` ngoài overlay dialog; không `linear-gradient` trong code UI (ngoại trừ skeleton shimmer).
- [ ] Không có card lồng card (review thủ công theo checklist component).
- [ ] 100% icon từ `lucide-react`, size 16/20, strokeWidth 1.75.
- [ ] Tất cả text ≥ 11px; body ≥ 13px; contrast AA pass bằng axe ở cả 2 theme.
- [ ] Office canvas nằm trong khung SaaS; overlay UI là DOM; shell không đổi style khi ở `/office`.
- [ ] Owner đã phê duyệt D2, D3, D7 trước khi bắt đầu Phase 1.
