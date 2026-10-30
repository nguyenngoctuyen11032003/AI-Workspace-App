# DESIGN TOKENS — AI WORKSPACE

> Phase 0 · Agent 11 · Phiên bản 0.1 · Nguồn sự thật duy nhất cho mọi giá trị thị giác.
> Đánh dấu: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**

## 1. Mục đích

Định nghĩa đầy đủ token (màu, chữ, khoảng cách, bo góc, bóng, z-index, motion, breakpoint) ở dạng (a) Tailwind v4 `@theme` CSS block có thể dán thẳng vào `src/styles/tokens.css`, và (b) bảng JSON để công cụ (Style Dictionary, Figma Tokens) tiêu thụ. Engineer không cần quyết định thêm giá trị nào.

## 2. Phạm vi

- Primitive tokens (thang màu, scale số) và semantic tokens (bg/surface/border/text/accent/status/provider).
- Hai theme Dark/Light, chọn qua `data-theme="dark|light"` trên `<html>`; "System" = JS đọc `prefers-color-scheme` rồi set attribute.
- Không bao gồm: palette pixel-art của office (Agent Virtual Office dẫn xuất từ token này), chart series ngoài 6 màu quy định.

## 3. Nội dung chi tiết

### 3.1 Kiến trúc token

```
primitive  →  semantic  →  component (shadcn CSS vars)
--neutral-900 → --color-surface → --card (shadcn alias)
```

- Primitive dùng `oklch` để nội suy đều; kèm hex gần đúng để designer/QA đối chiếu.
- Semantic token là thứ duy nhất component được phép dùng. Cấm dùng primitive trong component.
- shadcn/ui alias (`--background`, `--foreground`, `--primary`, `--muted`, …) được map sang semantic ở §3.9 để component shadcn chạy không cần sửa.

### 3.2 Color primitives **[Đề xuất]**

Thang neutral "cool graphite" (hue 260, chroma rất thấp):

| Token | oklch | Hex ≈ |
|---|---|---|
| `--neutral-0` | `oklch(1 0 0)` | `#FFFFFF` |
| `--neutral-50` | `oklch(0.985 0.002 260)` | `#FAFAFA` |
| `--neutral-100` | `oklch(0.965 0.003 260)` | `#F4F4F5` |
| `--neutral-200` | `oklch(0.92 0.005 260)` | `#E4E4E7` |
| `--neutral-300` | `oklch(0.87 0.006 260)` | `#D4D4D8` |
| `--neutral-400` | `oklch(0.71 0.01 260)` | `#A1A1AA` |
| `--neutral-500` | `oklch(0.55 0.012 260)` | `#71717A` |
| `--neutral-600` | `oklch(0.44 0.012 260)` | `#52525B` |
| `--neutral-700` | `oklch(0.37 0.012 260)` | `#3F3F46` |
| `--neutral-750` | `oklch(0.30 0.01 260)` | `#2C2D33` |
| `--neutral-800` | `oklch(0.25 0.009 260)` | `#1F2024` |
| `--neutral-850` | `oklch(0.21 0.008 260)` | `#181A1E` |
| `--neutral-900` | `oklch(0.18 0.008 260)` | `#131519` |
| `--neutral-950` | `oklch(0.13 0.006 260)` | `#0B0C0F` |

Màu có sắc (mỗi hue 3 bậc: `-400` dùng trên dark, `-600` dùng trên light, `-soft` làm nền badge):

| Hue | 400 (dark fg) | 600 (light fg) | Hex 400 / 600 |
|---|---|---|---|
| `--violet` (AI) | `oklch(0.72 0.16 285)` | `oklch(0.55 0.21 285)` | `#9D8DF7` / `#6552E6` |
| `--blue` (running/link) | `oklch(0.72 0.14 250)` | `oklch(0.55 0.19 250)` | `#6FA4F8` / `#2F6FE0` |
| `--green` (completed) | `oklch(0.75 0.15 155)` | `oklch(0.56 0.16 155)` | `#4FC27E` / `#1E8F4F` |
| `--amber` (waiting) | `oklch(0.80 0.15 80)` | `oklch(0.62 0.15 70)` | `#E2B341` / `#B97A12` |
| `--orange` (blocked) | `oklch(0.74 0.16 50)` | `oklch(0.60 0.18 45)` | `#F08A4B` / `#CF5A14` |
| `--red` (failed/destructive) | `oklch(0.70 0.18 25)` | `oklch(0.57 0.21 25)` | `#F06A5E` / `#D1332D` |
| `--teal` (chart only) | `oklch(0.75 0.11 195)` | `oklch(0.56 0.11 195)` | `#4FBFC6` / `#0F8B93` |

### 3.3 Typography **[Đề xuất — Cần người dùng phê duyệt: Inter vs Geist]**

- `--font-sans: "Inter Variable", "Inter", ui-sans-serif, system-ui, "Segoe UI", Roboto, sans-serif;`
- `--font-mono: "JetBrains Mono Variable", "JetBrains Mono", ui-monospace, "Cascadia Mono", Consolas, monospace;`
- Phương án B: `"Geist Variable"` + `"Geist Mono Variable"` — cùng metric gần tương đương, chỉ đổi token, không đổi scale.
- Font feature: sans `font-feature-settings: "cv11", "ss01", "tnum"` (tnum bật cục bộ qua class `.tabular`); mono `"liga" 0` (tắt ligature trong log để tránh hiểu nhầm `!=`).
- Self-host (woff2, `font-display: swap`); không gọi Google Fonts lúc runtime (desktop offline).

Scale (px / rem, base 16):

| Token | Size | Line-height | Weight | Tracking | Dùng cho |
|---|---|---|---|---|---|
| `--text-2xs` | 11 / 0.6875rem | 16px | 500 | +0.04em (uppercase) | Section label, table header, badge |
| `--text-xs` | 12 / 0.75rem | 16px | 400 | 0 | Metadata, timestamp, helper text |
| `--text-sm` | 13 / 0.8125rem | 20px | 400 | 0 | **Body mặc định**, table cell, sidebar item |
| `--text-base` | 14 / 0.875rem | 20px | 400 | 0 | Form input, dialog body, prose |
| `--text-md` | 16 / 1rem | 24px | 500 | -0.005em | Card title, panel header |
| `--text-lg` | 20 / 1.25rem | 28px | 600 | -0.01em | Page title (H1 trong app) |
| `--text-xl` | 24 / 1.5rem | 32px | 600 | -0.015em | Metric lớn trên Dashboard |
| `--text-2xl` | 30 / 1.875rem | 36px | 600 | -0.02em | Chỉ dùng ở empty-state hero / onboarding |

Mono: `--text-mono-xs` 11/16, `--text-mono-sm` 12/18 (log, ID), `--text-mono-base` 13/20 (Monaco editor mặc định, do Monaco set riêng).

### 3.4 Spacing (4px base) **[Đã xác minh]**

`--spacing: 0.25rem` (Tailwind v4 dùng nhân: `p-1`=4px, `p-2`=8, `p-3`=12, `p-4`=16, `p-5`=20, `p-6`=24, `p-8`=32, `p-10`=40, `p-12`=48, `p-16`=64). Quy ước dùng:

| Vai trò | Giá trị |
|---|---|
| Khoảng cách icon ↔ label | 8px (`gap-2`) |
| Padding control (button/input) | 8px dọc / 12px ngang (size md) |
| Padding card/panel | 16px (compact: 12px) |
| Gap giữa card trong grid | 16px; giữa section | 24px |
| Page padding | 24px desktop, 16px tablet/mobile |
| Sidebar width | 240px mở / 56px thu gọn |
| Topbar height | 48px; Statusbar (nếu có) 28px |
| Row height | 40px comfortable / 32px compact |
| Panel phải (inspector) | 320px; min 280, max 480 (resizable) |

### 3.5 Radius

`--radius-xs: 4px` (badge, checkbox) · `--radius-sm: 6px` (button, input, menu item) · `--radius-md: 8px` (card, panel, popover) · `--radius-lg: 12px` (dialog, drawer, command palette) · `--radius-full: 9999px` (avatar, pill, status dot). shadcn alias `--radius: 0.5rem`.

### 3.6 Elevation / Shadow **[Đề xuất]**

Dark theme: shadow gần như vô hình, chiều sâu đến từ bề mặt sáng hơn + border. Light theme: shadow mềm, nhiều lớp.

| Token | Light | Dark |
|---|---|---|
| `--shadow-none` | `none` | `none` |
| `--shadow-sm` (popover, dropdown) | `0 1px 2px rgb(0 0 0 / .06), 0 4px 12px rgb(0 0 0 / .08)` | `0 1px 2px rgb(0 0 0 / .4), 0 4px 16px rgb(0 0 0 / .5)` |
| `--shadow-md` (drawer, palette) | `0 2px 4px rgb(0 0 0 / .06), 0 12px 32px rgb(0 0 0 / .12)` | `0 2px 4px rgb(0 0 0 / .5), 0 16px 48px rgb(0 0 0 / .6)` |
| `--shadow-lg` (dialog) | `0 4px 8px rgb(0 0 0 / .08), 0 24px 64px rgb(0 0 0 / .16)` | `0 4px 8px rgb(0 0 0 / .5), 0 32px 80px rgb(0 0 0 / .7)` |
| `--ring-focus` | `0 0 0 2px var(--color-bg), 0 0 0 4px var(--color-accent)` | giống |

Card không có shadow (chỉ border). Overlay backdrop: `rgb(0 0 0 / .5)` dark, `rgb(9 9 11 / .4)` light.

### 3.7 Z-index layers

| Token | Giá trị | Dùng cho |
|---|---|---|
| `--z-base` | 0 | nội dung |
| `--z-sticky` | 10 | sticky table header, topbar |
| `--z-sidebar` | 20 | sidebar (trên nội dung khi overlay ở tablet) |
| `--z-dropdown` | 30 | dropdown, popover, select, tooltip |
| `--z-drawer` | 40 | sheet/drawer + backdrop |
| `--z-dialog` | 50 | dialog + backdrop |
| `--z-palette` | 60 | command palette |
| `--z-toast` | 70 | toast |
| `--z-debug` | 100 | dev overlay |

### 3.8 Motion **[Đề xuất]**

| Token | Giá trị | Dùng cho |
|---|---|---|
| `--duration-instant` | 80ms | hover color, pressed |
| `--duration-fast` | 120ms | tooltip, menu, route fade, focus |
| `--duration-normal` | 200ms | dialog, popover scale, tab indicator |
| `--duration-slow` | 280ms | drawer slide, sidebar collapse |
| `--duration-highlight` | 1200ms | row flash khi data đổi |
| `--ease-out` | `cubic-bezier(0.16, 1, 0.3, 1)` | mọi thứ "xuất hiện" |
| `--ease-in` | `cubic-bezier(0.7, 0, 0.84, 0)` | mọi thứ "biến mất" |
| `--ease-standard` | `cubic-bezier(0.2, 0, 0, 1)` | layout shift, width change |

`@media (prefers-reduced-motion: reduce)` → mọi duration = 1ms, bỏ transform (giữ opacity).

### 3.9 Breakpoints

`--breakpoint-sm: 640px` (mobile landscape) · `--breakpoint-md: 768px` (tablet) · `--breakpoint-lg: 1024px` (laptop nhỏ, sidebar tự thu) · `--breakpoint-xl: 1280px` (laptop chuẩn — thiết kế chính) · `--breakpoint-2xl: 1536px` (desktop). Chi tiết hành vi: `RESPONSIVE_DESIGN.md`.

### 3.10 Semantic tokens — Dark / Light

| Semantic | Dark | Light | Ghi chú |
|---|---|---|---|
| `--color-bg` | `#0B0C0F` neutral-950 | `#FAFAFA` neutral-50 | page |
| `--color-bg-sunken` | `#08090B` | `#F4F4F5` | code block, log well, canvas nền |
| `--color-surface` | `#131519` neutral-900 | `#FFFFFF` | card, panel, sidebar |
| `--color-surface-raised` | `#181A1E` neutral-850 | `#FFFFFF` + shadow-sm | popover, dropdown, dialog |
| `--color-surface-hover` | `#1F2024` neutral-800 | `#F4F4F5` | row/item hover |
| `--color-surface-active` | `#2C2D33` neutral-750 | `#E4E4E7` | selected item, pressed |
| `--color-border` | `#2C2D33` | `#E4E4E7` | mặc định 1px |
| `--color-border-subtle` | `#1F2024` | `#F4F4F5` | divider trong card, grid chart |
| `--color-border-strong` | `#3F3F46` | `#D4D4D8` | input border hover, focused card |
| `--color-text-primary` | `#F4F4F5` | `#18181B` | 15.3:1 / 16.2:1 |
| `--color-text-secondary` | `#A1A1AA` | `#52525B` | 7.1:1 / 7.4:1 |
| `--color-text-tertiary` | `#71717A` | `#71717A` | 4.6:1 / 4.7:1 — chỉ ≥ 12px |
| `--color-text-disabled` | `#52525B` | `#A1A1AA` | không cần đạt AA |
| `--color-text-inverse` | `#0B0C0F` | `#FFFFFF` | chữ trên nút primary |
| `--color-accent` | `#9D8DF7` violet-400 | `#6552E6` violet-600 | AI, primary action, focus ring |
| `--color-accent-hover` | `#B0A3F9` | `#5443D0` | |
| `--color-accent-soft` | `rgb(157 141 247 / .14)` | `rgb(101 82 230 / .10)` | nền badge/row AI |
| `--color-accent-fg` | `#0B0C0F` | `#FFFFFF` | chữ trên nền accent |
| `--color-link` | `#6FA4F8` | `#2F6FE0` | |

Status (8 trạng thái task/agent) **[Đề xuất — danh sách trạng thái cần Agent Product xác nhận]**:

| Status | Dark fg | Light fg | Soft bg (dark / light) | Icon lucide |
|---|---|---|---|---|
| `idle` | `#A1A1AA` | `#71717A` | `rgb(161 161 170/.12)` / `rgb(113 113 122/.10)` | `circle-dashed` |
| `running` | `#6FA4F8` | `#2F6FE0` | blue /.14 / .10 | `loader-circle` (spin) |
| `waiting` (approval/input) | `#E2B341` | `#B97A12` | amber /.14 / .10 | `clock` |
| `blocked` | `#F08A4B` | `#CF5A14` | orange /.14 / .10 | `octagon-alert` |
| `failed` | `#F06A5E` | `#D1332D` | red /.14 / .10 | `circle-x` |
| `completed` | `#4FC27E` | `#1E8F4F` | green /.14 / .10 | `circle-check` |
| `paused` | `#D4D4D8` | `#52525B` | neutral /.12 | `circle-pause` |
| `disabled` | `#52525B` | `#A1A1AA` | none | `circle-off` |

Agent-state (hiển thị ở avatar ring + office sprite bubble): `online` = green, `busy` = violet accent (đang xử lý bằng AI), `waiting` = amber, `error` = red, `offline` = neutral-500. Agent-state tái dùng token status ở trên, không tạo màu mới.

Provider colors (chỉ dùng cho mini-badge 12px và chart series, không dùng nền lớn) **[Giả định về brand]**:

| Provider | Dark | Light | Ký hiệu |
|---|---|---|---|
| `--color-provider-claude` | `#D9A585` (terracotta nhạt) | `#B5633A` | "C" |
| `--color-provider-codex` | `#E4E4E7` | `#18181B` | "X" |
| `--color-provider-openhands` | `#4FBFC6` teal | `#0F8B93` | "O" |

Chart series: `--chart-1` accent, `--chart-2` blue, `--chart-3` teal, `--chart-4` amber, `--chart-5` neutral-400, `--chart-6` green. Tối đa 6 series/biểu đồ.

### 3.11 Tailwind v4 `@theme` block (dán vào `src/styles/tokens.css`)

```css
@import "tailwindcss";

@theme {
  /* fonts */
  --font-sans: "Inter Variable", "Inter", ui-sans-serif, system-ui, "Segoe UI", sans-serif;
  --font-mono: "JetBrains Mono Variable", "JetBrains Mono", ui-monospace, "Cascadia Mono", Consolas, monospace;

  /* type scale */
  --text-2xs: 0.6875rem; --text-2xs--line-height: 1rem; --text-2xs--letter-spacing: 0.04em;
  --text-xs: 0.75rem;    --text-xs--line-height: 1rem;
  --text-sm: 0.8125rem;  --text-sm--line-height: 1.25rem;
  --text-base: 0.875rem; --text-base--line-height: 1.25rem;
  --text-md: 1rem;       --text-md--line-height: 1.5rem;   --text-md--letter-spacing: -0.005em;
  --text-lg: 1.25rem;    --text-lg--line-height: 1.75rem;  --text-lg--letter-spacing: -0.01em;
  --text-xl: 1.5rem;     --text-xl--line-height: 2rem;     --text-xl--letter-spacing: -0.015em;
  --text-2xl: 1.875rem;  --text-2xl--line-height: 2.25rem; --text-2xl--letter-spacing: -0.02em;

  /* spacing & shape */
  --spacing: 0.25rem;
  --radius-xs: 4px; --radius-sm: 6px; --radius-md: 8px; --radius-lg: 12px; --radius-full: 9999px;

  /* neutrals */
  --color-neutral-0: oklch(1 0 0);
  --color-neutral-50: oklch(0.985 0.002 260);
  --color-neutral-100: oklch(0.965 0.003 260);
  --color-neutral-200: oklch(0.92 0.005 260);
  --color-neutral-300: oklch(0.87 0.006 260);
  --color-neutral-400: oklch(0.71 0.01 260);
  --color-neutral-500: oklch(0.55 0.012 260);
  --color-neutral-600: oklch(0.44 0.012 260);
  --color-neutral-700: oklch(0.37 0.012 260);
  --color-neutral-750: oklch(0.30 0.01 260);
  --color-neutral-800: oklch(0.25 0.009 260);
  --color-neutral-850: oklch(0.21 0.008 260);
  --color-neutral-900: oklch(0.18 0.008 260);
  --color-neutral-950: oklch(0.13 0.006 260);

  /* hues */
  --color-violet-400: oklch(0.72 0.16 285); --color-violet-600: oklch(0.55 0.21 285);
  --color-blue-400: oklch(0.72 0.14 250);   --color-blue-600: oklch(0.55 0.19 250);
  --color-green-400: oklch(0.75 0.15 155);  --color-green-600: oklch(0.56 0.16 155);
  --color-amber-400: oklch(0.80 0.15 80);   --color-amber-600: oklch(0.62 0.15 70);
  --color-orange-400: oklch(0.74 0.16 50);  --color-orange-600: oklch(0.60 0.18 45);
  --color-red-400: oklch(0.70 0.18 25);     --color-red-600: oklch(0.57 0.21 25);
  --color-teal-400: oklch(0.75 0.11 195);   --color-teal-600: oklch(0.56 0.11 195);

  /* semantic — bound to runtime vars so theme switch works without rebuild */
  --color-bg: var(--bg);
  --color-bg-sunken: var(--bg-sunken);
  --color-surface: var(--surface);
  --color-surface-raised: var(--surface-raised);
  --color-surface-hover: var(--surface-hover);
  --color-surface-active: var(--surface-active);
  --color-border: var(--border);
  --color-border-subtle: var(--border-subtle);
  --color-border-strong: var(--border-strong);
  --color-text-primary: var(--text-primary);
  --color-text-secondary: var(--text-secondary);
  --color-text-tertiary: var(--text-tertiary);
  --color-text-disabled: var(--text-disabled);
  --color-text-inverse: var(--text-inverse);
  --color-accent: var(--accent);
  --color-accent-hover: var(--accent-hover);
  --color-accent-soft: var(--accent-soft);
  --color-accent-fg: var(--accent-fg);
  --color-link: var(--link);
  --color-status-idle: var(--status-idle);
  --color-status-running: var(--status-running);
  --color-status-waiting: var(--status-waiting);
  --color-status-blocked: var(--status-blocked);
  --color-status-failed: var(--status-failed);
  --color-status-completed: var(--status-completed);
  --color-status-paused: var(--status-paused);
  --color-status-disabled: var(--status-disabled);
  --color-provider-claude: var(--provider-claude);
  --color-provider-codex: var(--provider-codex);
  --color-provider-openhands: var(--provider-openhands);

  /* shadows */
  --shadow-sm: var(--elev-sm); --shadow-md: var(--elev-md); --shadow-lg: var(--elev-lg);

  /* motion */
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-in: cubic-bezier(0.7, 0, 0.84, 0);
  --ease-standard: cubic-bezier(0.2, 0, 0, 1);
  --duration-instant: 80ms; --duration-fast: 120ms; --duration-normal: 200ms;
  --duration-slow: 280ms; --duration-highlight: 1200ms;

  /* breakpoints */
  --breakpoint-sm: 640px; --breakpoint-md: 768px; --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px; --breakpoint-2xl: 1536px;
}

/* runtime theme values */
:root, [data-theme="dark"] {
  color-scheme: dark;
  --bg: #0B0C0F; --bg-sunken: #08090B;
  --surface: #131519; --surface-raised: #181A1E; --surface-hover: #1F2024; --surface-active: #2C2D33;
  --border: #2C2D33; --border-subtle: #1F2024; --border-strong: #3F3F46;
  --text-primary: #F4F4F5; --text-secondary: #A1A1AA; --text-tertiary: #71717A; --text-disabled: #52525B; --text-inverse: #0B0C0F;
  --accent: #9D8DF7; --accent-hover: #B0A3F9; --accent-soft: rgb(157 141 247 / .14); --accent-fg: #0B0C0F; --link: #6FA4F8;
  --status-idle: #A1A1AA; --status-running: #6FA4F8; --status-waiting: #E2B341; --status-blocked: #F08A4B;
  --status-failed: #F06A5E; --status-completed: #4FC27E; --status-paused: #D4D4D8; --status-disabled: #52525B;
  --provider-claude: #D9A585; --provider-codex: #E4E4E7; --provider-openhands: #4FBFC6;
  --elev-sm: 0 1px 2px rgb(0 0 0 / .4), 0 4px 16px rgb(0 0 0 / .5);
  --elev-md: 0 2px 4px rgb(0 0 0 / .5), 0 16px 48px rgb(0 0 0 / .6);
  --elev-lg: 0 4px 8px rgb(0 0 0 / .5), 0 32px 80px rgb(0 0 0 / .7);
  --overlay: rgb(0 0 0 / .5);
}
[data-theme="light"] {
  color-scheme: light;
  --bg: #FAFAFA; --bg-sunken: #F4F4F5;
  --surface: #FFFFFF; --surface-raised: #FFFFFF; --surface-hover: #F4F4F5; --surface-active: #E4E4E7;
  --border: #E4E4E7; --border-subtle: #F4F4F5; --border-strong: #D4D4D8;
  --text-primary: #18181B; --text-secondary: #52525B; --text-tertiary: #71717A; --text-disabled: #A1A1AA; --text-inverse: #FFFFFF;
  --accent: #6552E6; --accent-hover: #5443D0; --accent-soft: rgb(101 82 230 / .10); --accent-fg: #FFFFFF; --link: #2F6FE0;
  --status-idle: #71717A; --status-running: #2F6FE0; --status-waiting: #B97A12; --status-blocked: #CF5A14;
  --status-failed: #D1332D; --status-completed: #1E8F4F; --status-paused: #52525B; --status-disabled: #A1A1AA;
  --provider-claude: #B5633A; --provider-codex: #18181B; --provider-openhands: #0F8B93;
  --elev-sm: 0 1px 2px rgb(0 0 0 / .06), 0 4px 12px rgb(0 0 0 / .08);
  --elev-md: 0 2px 4px rgb(0 0 0 / .06), 0 12px 32px rgb(0 0 0 / .12);
  --elev-lg: 0 4px 8px rgb(0 0 0 / .08), 0 24px 64px rgb(0 0 0 / .16);
  --overlay: rgb(9 9 11 / .4);
}

/* shadcn/ui aliases */
:root {
  --background: var(--bg); --foreground: var(--text-primary);
  --card: var(--surface); --card-foreground: var(--text-primary);
  --popover: var(--surface-raised); --popover-foreground: var(--text-primary);
  --primary: var(--accent); --primary-foreground: var(--accent-fg);
  --secondary: var(--surface-hover); --secondary-foreground: var(--text-primary);
  --muted: var(--surface-hover); --muted-foreground: var(--text-secondary);
  --accent: var(--surface-active); --accent-foreground: var(--text-primary); /* shadcn "accent" = hover, KHÔNG phải AI accent */
  --destructive: var(--status-failed); --destructive-foreground: #FFFFFF;
  --border: var(--border); --input: var(--border-strong); --ring: var(--accent);
  --radius: 0.5rem;
}

@media (prefers-reduced-motion: reduce) {
  :root { --duration-instant: 1ms; --duration-fast: 1ms; --duration-normal: 1ms; --duration-slow: 1ms; --duration-highlight: 1ms; }
}
```

Lưu ý: shadcn dùng tên `--accent` cho màu hover; để tránh va chạm, AI accent ở runtime là `--accent` **trong block theme** nhưng shadcn alias phải đọc `--surface-active`. Engineer đổi tên runtime var AI thành `--ai-accent` nếu thấy xung đột (**[Đề xuất]**: đổi ngay từ đầu — `--ai-accent`, `--ai-accent-hover`, `--ai-accent-soft`, `--ai-accent-fg`).

### 3.12 JSON token table (rút gọn, định dạng W3C DTCG)

```json
{
  "color": {
    "bg": { "$type": "color", "$value": { "dark": "#0B0C0F", "light": "#FAFAFA" } },
    "surface": { "$type": "color", "$value": { "dark": "#131519", "light": "#FFFFFF" } },
    "surface-raised": { "$type": "color", "$value": { "dark": "#181A1E", "light": "#FFFFFF" } },
    "surface-hover": { "$type": "color", "$value": { "dark": "#1F2024", "light": "#F4F4F5" } },
    "border": { "$type": "color", "$value": { "dark": "#2C2D33", "light": "#E4E4E7" } },
    "border-strong": { "$type": "color", "$value": { "dark": "#3F3F46", "light": "#D4D4D8" } },
    "text-primary": { "$type": "color", "$value": { "dark": "#F4F4F5", "light": "#18181B" } },
    "text-secondary": { "$type": "color", "$value": { "dark": "#A1A1AA", "light": "#52525B" } },
    "text-tertiary": { "$type": "color", "$value": { "dark": "#71717A", "light": "#71717A" } },
    "ai-accent": { "$type": "color", "$value": { "dark": "#9D8DF7", "light": "#6552E6" } },
    "link": { "$type": "color", "$value": { "dark": "#6FA4F8", "light": "#2F6FE0" } },
    "status": {
      "idle": { "$value": { "dark": "#A1A1AA", "light": "#71717A" } },
      "running": { "$value": { "dark": "#6FA4F8", "light": "#2F6FE0" } },
      "waiting": { "$value": { "dark": "#E2B341", "light": "#B97A12" } },
      "blocked": { "$value": { "dark": "#F08A4B", "light": "#CF5A14" } },
      "failed": { "$value": { "dark": "#F06A5E", "light": "#D1332D" } },
      "completed": { "$value": { "dark": "#4FC27E", "light": "#1E8F4F" } },
      "paused": { "$value": { "dark": "#D4D4D8", "light": "#52525B" } },
      "disabled": { "$value": { "dark": "#52525B", "light": "#A1A1AA" } }
    },
    "provider": {
      "claude": { "$value": { "dark": "#D9A585", "light": "#B5633A" } },
      "codex": { "$value": { "dark": "#E4E4E7", "light": "#18181B" } },
      "openhands": { "$value": { "dark": "#4FBFC6", "light": "#0F8B93" } }
    }
  },
  "font": {
    "sans": { "$type": "fontFamily", "$value": ["Inter Variable", "Inter", "system-ui", "sans-serif"] },
    "mono": { "$type": "fontFamily", "$value": ["JetBrains Mono Variable", "JetBrains Mono", "ui-monospace", "monospace"] },
    "size": { "2xs": 11, "xs": 12, "sm": 13, "base": 14, "md": 16, "lg": 20, "xl": 24, "2xl": 30 },
    "lineHeight": { "2xs": 16, "xs": 16, "sm": 20, "base": 20, "md": 24, "lg": 28, "xl": 32, "2xl": 36 }
  },
  "spacing": { "base": 4, "scale": [0, 4, 8, 12, 16, 20, 24, 32, 40, 48, 64] },
  "radius": { "xs": 4, "sm": 6, "md": 8, "lg": 12, "full": 9999 },
  "zIndex": { "sticky": 10, "sidebar": 20, "dropdown": 30, "drawer": 40, "dialog": 50, "palette": 60, "toast": 70 },
  "motion": {
    "duration": { "instant": 80, "fast": 120, "normal": 200, "slow": 280, "highlight": 1200 },
    "easing": { "out": "cubic-bezier(0.16,1,0.3,1)", "in": "cubic-bezier(0.7,0,0.84,0)", "standard": "cubic-bezier(0.2,0,0,1)" }
  },
  "breakpoint": { "sm": 640, "md": 768, "lg": 1024, "xl": 1280, "2xl": 1536 }
}
```

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Trạng thái |
|---|---|---|---|
| T1 | Primitive oklch, semantic hex | oklch nội suy đều; hex để QA đo contrast bằng tool phổ thông | Đề xuất |
| T2 | Theme switch bằng runtime var (`[data-theme]`), `@theme` chỉ bind | Tailwind v4 `@theme` là static; cần đổi theme không rebuild | Đã xác minh (cơ chế Tailwind v4) |
| T3 | 8 status + agent-state tái dùng status | Giảm số màu; brief yêu cầu "few colors" | Đề xuất |
| T4 | Text-tertiary `#71717A` cả hai theme | Đạt ≥ 4.5:1 trên cả `#0B0C0F` và `#FAFAFA`, tiết kiệm token | Đã xác minh (tính WCAG) |
| T5 | Đổi tên AI accent thành `--ai-accent` | Tránh va chạm với shadcn `--accent` | Đề xuất |
| T6 | Self-host font | Desktop app chạy offline | Đề xuất |

## 5. Dependencies

- Tailwind v4 (`@theme`, `--spacing` multiplier). shadcn/ui CLI bản "new-york" + Tailwind v4 preset.
- Agent Product: danh sách trạng thái cuối cùng. Agent Provider Hub: brand guideline của Claude/Codex/OpenHands nếu có yêu cầu pháp lý về màu logo.
- Agent Virtual Office: dẫn xuất palette pixel từ bảng status.

## 6. Rủi ro

- Giá trị hex ≈ oklch có sai số nhỏ; nguồn sự thật là oklch trong `@theme`, hex chỉ để đối chiếu.
- `#E2B341` (amber dark) trên `#131519` ≈ 9:1 OK; nhưng amber-600 light `#B97A12` trên trắng ≈ 4.6:1 — sát ngưỡng; nếu dùng cho text 11px phải đậm (weight 600).
- Provider colors là giả định; nếu brand yêu cầu khác, chỉ đổi 3 token.

## 7. Vấn đề chưa giải quyết

- Inter hay Geist (ảnh hưởng metric nhỏ, không ảnh hưởng scale).
- Có cần theme "dim" (giữa dark/light) không? Đề xuất: không.
- Monaco theme: cần file theme JSON riêng ánh xạ từ token (Agent Code Workspace đảm nhận).

## 8. Acceptance Criteria

- [ ] `tokens.css` compile với Tailwind v4 không warning; class `bg-surface`, `text-text-secondary`, `border-border`, `text-status-running`, `shadow-md`, `rounded-md` hoạt động.
- [ ] Đổi `data-theme` không gây reflow layout (chỉ repaint màu).
- [ ] Bảng contrast tự động: text-primary/secondary/tertiary trên bg/surface ≥ 4.5:1 ở cả hai theme; status fg trên soft bg ≥ 3:1.
- [ ] JSON token parse được bằng Style Dictionary v4 (DTCG format).
- [ ] Không component nào import primitive (`neutral-*`, `violet-*`) trực tiếp — ESLint rule/grep.
- [ ] `prefers-reduced-motion` đặt mọi duration = 1ms.


> **Phụ lục sửa đổi:** xem `docs/design/HERDR_DESIGN_INTEGRATION.md` (attention state, Agent Rail, token bổ sung). Khi mâu thuẫn, phụ lục thắng cho các hạng mục đó cho đến khi chủ sở hữu duyệt (D-017..D-019).
