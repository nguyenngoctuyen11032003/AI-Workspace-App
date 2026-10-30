# COMPONENT SPECIFICATION — Catalogue component và hợp đồng props

Trạng thái: DRAFT (Phase 0). Ngày: 2026-10-30. Tác giả: Delivery/Engineering Lead.
Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].
Nguồn trực quan (không lặp lại ở đây): `DESIGN_SYSTEM.md` (kích thước, màu, state hình ảnh), `DESIGN_TOKENS.md`, `INTERACTION_DESIGN.md`, `SCREEN_SPECIFICATION.md`, `NAVIGATION_STRUCTURE.md`, `RESPONSIVE_DESIGN.md`.
Liên quan: `../development/CODING_STANDARDS.md` §3.8, §3.11; `../development/TESTING.md` §3.4.

---

## 1. Mục đích

Đặc tả **hợp đồng lập trình** của từng component: props (TypeScript), variant, state, event, accessibility, test tối thiểu. `DESIGN_SYSTEM.md` trả lời "trông thế nào"; tài liệu này trả lời "gọi thế nào, phát ra gì, kiểm gì". Mục tiêu là để agent `frontend` triển khai độc lập, nhiều agent làm song song mà không lệch API component.

## 2. Phạm vi

- Trong: primitives (bọc shadcn/Radix), composite (ghép primitives), domain components (AgentChip, TaskCard, ApprovalCard, WorkflowNode, LogViewer, OfficeHUD, ...).
- Ngoài: giá trị token/kích thước (xem DESIGN_SYSTEM/TOKENS), layout từng màn (SCREEN_SPECIFICATION), logic gọi API (feature layer), canvas Phaser bên trong Virtual Office (chỉ đặc tả HUD/panel React bao quanh).
- Chưa có mã; mọi interface là thiết kế Phase 1-5. [Đã xác minh]

## 3. Nội dung chi tiết

### 3.0 Quy ước chung [Đề xuất]

Kế thừa `DESIGN_SYSTEM.md` §3.0 (thư mục `components/ui` = shadcn, `components/app` = custom; `className` + forward ref; `cva` + `cn()`; size `sm|md|lg`; focus ring; disabled; icon aria).

```ts
// packages/ui/src/common.ts (hoặc apps/web/src/components/common.ts)
export type Size = 'sm' | 'md' | 'lg';
export interface BaseProps { className?: string; 'data-testid'?: string }
export type Density = 'comfortable' | 'compact';

/** Trạng thái hiển thị thống nhất, ánh xạ từ domain state (xem 3.0.1). */
export type StatusTone =
  | 'idle' | 'running' | 'waiting' | 'blocked'
  | 'failed' | 'completed' | 'paused' | 'disabled';

export type AsyncState = 'ready' | 'loading' | 'empty' | 'error' | 'offline';
```

**3.0.1 Ánh xạ domain state → `StatusTone`.** Có ba bộ trạng thái khác nhau trong tài liệu hiện có: 12 trạng thái lifecycle agent (FR-024), 10 trạng thái task (FR-108), 8 tone trực quan (`DESIGN_SYSTEM.md` §3.8). Component **không** nhận state domain thô mà nhận `StatusTone` đã được map qua hàm thuần `toTone(entityType, state)` đặt ở `packages/contracts` (một nguồn). Bảng map đầy đủ chờ Architect (xem Vấn đề chưa giải quyết 1); đề xuất:

| Domain state | Tone |
|---|---|
| agent: Created/Idle/Available · task: BACKLOG/READY | `idle` |
| agent: Assigned/Running · task: ASSIGNED/RUNNING/REVIEWING | `running` |
| agent: Waiting* · task: WAITING_APPROVAL | `waiting` |
| agent/task: Blocked | `blocked` |
| agent/task: Failed/Error | `failed` |
| task: COMPLETED/DONE | `completed` |
| agent: Paused | `paused` |
| agent: Disabled/Archived | `disabled` |

**3.0.2 Quy tắc chung cho mọi component** (kiểm bằng test component):
1. Không fetch dữ liệu; nhận dữ liệu qua props, phát sự kiện qua callback `onX`. Ngoại lệ: component "container" nằm trong `features/`.
2. Không chứa dữ liệu mẫu/hardcode agent hoặc workflow (CODING_STANDARDS F-01..F-03). Story/test dùng fixture từ `packages/testing`.
3. Có đủ `AsyncState` khi hiển thị danh sách/dữ liệu từ xa (dùng `StateBlock`).
4. Không nhận hoặc hiển thị credential; props chứa `providerId`, không chứa token (F-04).
5. Hiển thị số liệu ước tính phải truyền `estimated: true` để UI gắn nhãn (F-19).
6. i18n: mọi chuỗi qua `t()`; props nhận `labels?: Partial<Labels>` chỉ khi cần ghi đè.
7. `prefers-reduced-motion` tắt spin/pulse/dash (xem DESIGN_SYSTEM).
8. Keyboard: mọi hành động bằng chuột có tương đương bàn phím; thứ tự Tab theo thứ tự đọc.
9. Test tối thiểu: render từng variant, từng state, sự kiện chính, `axe` không vi phạm.

### 3.1 Nhóm Primitives (bọc shadcn/Radix) [Đề xuất]

Các primitive giữ API shadcn; bảng chỉ liệt kê phần **mở rộng của dự án** và quy tắc. Chi tiết visual: DESIGN_SYSTEM §3.1-3.16.

| Component | Nguồn | Variant | Mở rộng/quy tắc | State | A11y |
|---|---|---|---|---|---|
| `Button` | shadcn | `primary, secondary, ghost, outline, destructive, link`; size `sm, md, lg, icon, icon-sm` | thêm `loading`, `leftIcon`, `rightIcon`, `shortcut` | default, hover, focus-visible, active, disabled, loading | icon-only bắt buộc `aria-label` + Tooltip; `loading` đặt `aria-busy` |
| `Input`, `Textarea` | shadcn | size; `invalid` | `prefix/suffix`, `tokenCounter` (Textarea) | default, focus, error, disabled, readonly | `aria-invalid`, `aria-describedby` tới helper |
| `Select`, `Combobox` | shadcn/Radix, `Command` | — | Combobox khi > 8 lựa chọn; `loadOptions` async (ListModels) | closed, open, loading, empty, error | `role=combobox`, `aria-expanded`, điều hướng ↑↓/Enter/Esc |
| `Checkbox`, `Switch`, `RadioGroup` | shadcn | — | — | checked/unchecked/indeterminate/disabled | nhãn liên kết `htmlFor` |
| `DropdownMenu`, `ContextMenu` | shadcn | — | item có `shortcut`, `destructive` | open/closed | roving tabindex, Esc đóng |
| `Tooltip`, `Popover`, `HoverCard` | shadcn | — | delay 400/—/600 ms (DESIGN_SYSTEM §3.4) | open/closed | tooltip không chứa nội dung tương tác; đóng bằng Esc |
| `Dialog`, `AlertDialog` | shadcn | width `sm, md, lg, xl` | `AlertDialog` cho hành động phá huỷ, hỗ trợ `confirmText` gõ lại (High risk) | open/closed/submitting | focus trap, trả focus về trigger, `aria-labelledby/describedby` |
| `Sheet` | shadcn | width `md, lg, xl` | `pinnable` (split ≥ xl) | open/closed/pinned | như Dialog |
| `Tabs` | shadcn | `line, segmented` | `urlSync` (đồng bộ `?tab=`) | active/inactive/disabled | `role=tablist`, ←/→ |
| `Badge` | shadcn | `neutral, outline, ai` | — | — | — |
| `Avatar` | shadcn | size 20/24/32/40/64 | `shape: 'circle' | 'rounded'` (người tròn, agent bo) | loaded/fallback | `alt` hoặc `aria-hidden` |
| `Card`, `Section` | shadcn/custom | `compact` | không lồng Card | default, clickable, selected | clickable card là `button`/`a` |
| `Skeleton` | shadcn | — | phải mô phỏng layout cuối | — | `aria-hidden`, vùng cha `aria-busy` |
| `Toast` (Sonner) | shadcn | `info, success, warning, error` | `action`, `duration`, `dedupeKey` | — | `role=status`/`alert` theo mức |
| `Progress` | shadcn | `xs, sm`; determinate/indeterminate | — | — | `role=progressbar`, `aria-valuenow` |
| `Kbd` | custom | — | — | — | `<kbd>` |
| `Resizable`, `ScrollArea`, `Separator`, `Accordion` | shadcn | — | — | — | theo Radix |

### 3.2 Nhóm Composite (ghép primitives, không gắn domain) [Đề xuất]

#### 3.2.1 `StatusBadge`

```ts
export interface StatusBadgeProps extends BaseProps {
  status: StatusTone;
  size?: 'sm' | 'md';
  showLabel?: boolean;          // false = dot-only (dùng trong AgentChip)
  label?: string;               // ghi đè nhãn đã i18n
  reason?: string;              // tooltip, bắt buộc khi status='blocked'
}
```
- Variants: 8 tone. State: `running` spin, `waiting` pulse (tắt khi reduced-motion).
- A11y: `role="status"` không cần (tĩnh); `aria-label="Status: running"`; không chỉ dùng màu (luôn dot/icon + chữ khi `showLabel`).
- Events: không.

#### 3.2.2 `AgentAvatar`

```ts
export type AgentPresence = 'online' | 'busy' | 'waiting' | 'error' | 'offline';
export interface AgentAvatarProps extends BaseProps {
  name: string;
  src?: string;                 // sprite/avatar đã được cấp phép (assets-manifest)
  size?: 20 | 24 | 32 | 40 | 64;
  presence?: AgentPresence;     // ring màu
  providerId?: ProviderKind;    // 'claude-code' | 'codex' | 'openhands' | 'generic-cli'
  ariaHidden?: boolean;         // khi đi cùng tên text
}
```
- Radius agent = `md` (D-015). Provider mini-badge chỉ khi `size >= 24`.
- State: loaded, fallback initials, ring theo `presence`.
- A11y: nếu không `ariaHidden`, `alt = "${name}, ${presence}"`.

#### 3.2.3 `StateBlock`

```ts
export interface StateBlockProps extends BaseProps {
  variant: 'empty' | 'error' | 'success' | 'offline';
  icon?: ReactNode;
  title: string;
  description?: string;
  errorCode?: string;           // hiển thị mono khi variant='error'
  correlationId?: string;
  action?: { label: string; onClick: () => void };
  secondaryAction?: { label: string; onClick: () => void };
}
```
- A11y: `role="alert"` khi `error`, `role="status"` còn lại; focus chuyển vào tiêu đề khi thay thế vùng nội dung. Luôn kèm CTA (SCREEN_SPECIFICATION §2).

#### 3.2.4 `DataTable<T>`

```ts
export interface DataTableProps<T> extends BaseProps {
  data: readonly T[];
  columns: ColumnDef<T>[];                       // TanStack Table
  getRowId: (row: T) => string;
  state?: AsyncState;
  density?: Density;
  selectionMode?: 'none' | 'single' | 'multi';
  sorting?: SortingState; onSortingChange?: (s: SortingState) => void;
  pagination?: { pageSize: number; cursor?: string; total?: number };
  onPageChange?: (cursor: string | undefined) => void;
  onRowOpen?: (row: T) => void;                  // Enter / double-click
  onSelectionChange?: (ids: string[]) => void;
  virtualizeThreshold?: number;                  // mặc định 100
  persistKey?: string;                           // lưu density/cột trong localStorage
}
```
- Events: `onRowOpen`, `onSelectionChange`, `onSortingChange`, `onPageChange`.
- Keyboard: ↑↓ di chuyển, Enter mở, Space chọn, Ctrl+A chọn hết (DESIGN_SYSTEM §3.11).
- A11y: `role="grid"`, `aria-sort`, `aria-rowcount`; header sticky; vùng live đọc "1–50 of 1,204".

#### 3.2.5 `ChartCard` / `Sparkline`

```ts
export interface SeriesPoint { t: string; v: number }   // t ISO-8601
export interface SparklineProps extends BaseProps { data: readonly SeriesPoint[]; height?: 24 | 32; tone?: StatusTone | 'accent' }
export interface ChartCardProps extends BaseProps {
  title: string; kind: 'line' | 'bar' | 'donut';
  series: ReadonlyArray<{ id: string; label: string; points: readonly SeriesPoint[] }>;
  state?: AsyncState; estimated?: boolean; unit?: 'tokens' | 'usd' | 'count' | 'ms';
}
```
- A11y: kèm bảng dữ liệu ẩn (`<table class="sr-only">`) hoặc `aria-describedby` tóm tắt; không chỉ dùng màu (≤ 3 series, có label đường).
- `estimated` hiển thị nhãn "est." (FR-096).

#### 3.2.6 `Sidebar`, `Topbar`, `Breadcrumb`, `CommandPalette`

```ts
export interface NavItem { id: string; label: string; icon: IconName; href: string; badge?: number; group: string }
export interface SidebarProps extends BaseProps { items: NavItem[]; collapsed: boolean; onCollapsedChange(v: boolean): void; projectSwitcher: ReactNode }
export interface CommandItem { id: string; group: 'Navigate' | 'Actions' | 'Agents' | 'Tasks'; label: string; keywords?: string[]; shortcut?: string; run: () => void }
export interface CommandPaletteProps { open: boolean; onOpenChange(v: boolean): void; items: CommandItem[]; onQueryChange?(q: string): void }
```
- `Sidebar` collapsed 56px: icon + Tooltip. `CommandPalette` mở bằng Ctrl/Cmd+K, `role=dialog` + combobox pattern, hành động phá huỷ phải qua xác nhận, không chạy thẳng.
- `Topbar` luôn chứa `EmergencyStopButton` (3.3.9) trên mọi màn (AC-112).

#### 3.2.7 `FormField` và `Stepper`

```ts
export interface FormFieldProps extends BaseProps { label: string; htmlFor: string; required?: boolean; help?: string; error?: string; children: ReactNode }
export interface StepperProps extends BaseProps { steps: ReadonlyArray<{ id: string; title: string; status: 'todo'|'current'|'done'|'error' }>; current: string; onStepChange?(id: string): void; allowJump?: boolean }
```
- Dùng cho Agent Builder (6 bước, SCREEN_SPECIFICATION §3.8), Teach Agent. A11y: `aria-current="step"`; lỗi gắn `aria-describedby`.

### 3.3 Nhóm Domain components [Đề xuất]

Các kiểu dữ liệu dùng chung (rút gọn, nguồn là `packages/contracts`):

```ts
export type ProviderKind = 'claude-code' | 'codex' | 'openhands' | 'generic-cli';
export interface AgentSummary {
  id: AgentId; name: string; roleTitle: string; departmentId: string;
  providerId: ProviderKind; model?: string; avatarSrc?: string;
  tone: StatusTone; currentActivity?: string;          // ≤ 24 ký tự khi hiển thị
}
export type Priority = 'P0' | 'P1' | 'P2' | 'P3';
```

#### 3.3.1 `AgentChip`

```ts
export interface AgentChipProps extends BaseProps {
  agent: Pick<AgentSummary, 'id' | 'name' | 'providerId' | 'avatarSrc' | 'tone' | 'currentActivity'>;
  size?: 'sm' | 'md';                 // h-24 / h-28
  showActivity?: boolean;             // mặc định false
  interactive?: boolean;              // mặc định true: hover -> HoverCard, click -> onOpen
  onOpen?: (id: AgentId) => void;
}
```
- Hình thức: `DESIGN_SYSTEM.md` §3.18. State: default, hover (HoverCard 600 ms), focus-visible, disabled (agent `disabled`), loading (skeleton pill), unknown (agent đã xoá mềm: gạch chân, tooltip "Archived").
- Events: `onOpen` (click/Enter). Không điều hướng trực tiếp.
- A11y: là `button`; tên accessible: "Nova, Claude Code, running"; `showActivity` thêm mô tả; HoverCard có thể mở bằng focus.
- Dùng trong: TaskCard, ApprovalCard, Topbar activity, WorkflowNode.

#### 3.3.2 `TaskCard` và `TaskRow`

```ts
export interface TaskView {
  id: TaskId; displayId: string; title: string; priority: Priority; tone: StatusTone;
  labels: string[]; assignee?: AgentSummary; elapsedMs?: number; dueAt?: string;
  dependencyCount: number; approvalPending: boolean; blockedReason?: string;
  assignmentMode?: 'AUTO' | 'MANUAL';
}
export interface TaskCardProps extends BaseProps {
  task: TaskView;
  selected?: boolean;
  draggable?: boolean;                // mặc định true trên Kanban
  density?: Density;
  onOpen?: (id: TaskId) => void;
  onMenuAction?: (id: TaskId, action: 'assign' | 'prioritize' | 'block' | 'cancel') => void;
  onDragStart?: (id: TaskId) => void;
  onDrop?: (id: TaskId, toColumn: string) => void;   // chỉ phát sự kiện; chuyển trạng thái do state machine ở BE quyết
}
export type TaskRowProps = Omit<TaskCardProps, 'draggable'> & { columns: readonly TaskColumnId[] };
```
- Hình thức: `DESIGN_SYSTEM.md` §3.19. State: default, hover (hiện menu), selected, dragging, drop-target, loading (skeleton), blocked (icon `octagon-alert` + tooltip `blockedReason`), approval-pending.
- **Quan trọng:** drop không đổi trạng thái cục bộ (optimistic) vì trạng thái chỉ đổi qua event (FR-025/118); UI hiển thị "pending" tới khi event `TASK_STATE_CHANGED` đến, hoặc rollback khi BE từ chối (mã lỗi).
- Keyboard: Enter mở, Space chọn, `Alt+↑/↓` di chuyển trong cột, `M` mở menu; kéo-thả có thay thế bàn phím (WCAG 2.5.7).
- A11y: `article` với `aria-label` "P1 task TASK-12, running, assigned to Nova"; live region báo khi tone đổi.

#### 3.3.3 `ApprovalCard`

```ts
export type ApprovalKind = 'code_change' | 'shell_command' | 'spend' | 'external_call' | 'stage_gate';
export type RiskLevel = 'low' | 'medium' | 'high';
export interface ApprovalView {
  id: ApprovalId; kind: ApprovalKind; risk: RiskLevel; requestedAt: string;
  summary: string; requester: AgentSummary; taskId?: TaskId;
  preview:
    | { type: 'diff'; lines: readonly string[]; truncated: boolean }
    | { type: 'command'; command: string; cwd: string }
    | { type: 'amount'; value: number; currency: 'USD' | 'tokens'; estimated: boolean }
    | { type: 'text'; text: string };
}
export interface ApprovalDecision { decision: 'approve' | 'reject' | 'request_changes'; feedback?: string; confirmText?: string }
export interface ApprovalCardProps extends BaseProps {
  approval: ApprovalView;
  submitting?: boolean;
  onDecide: (id: ApprovalId, d: ApprovalDecision) => Promise<void> | void;
  onViewDetails?: (id: ApprovalId) => void;
}
```
- Hình thức: `DESIGN_SYSTEM.md` §3.20. State: pending, submitting (nút loading, khoá), decided (hiển thị kết quả, readonly), expired/cancelled, error (giữ form, hiện `StateBlock` nhỏ).
- Quy tắc: `reject` và `request_changes` **bắt buộc `feedback`** (AC-046 [Đề xuất]); `risk='high'` cần `confirmText` khớp từ khoá (gõ xác nhận); phím tắt chỉ mở form, không quyết ngay.
- Events: `onDecide`, `onViewDetails`. Phát ra `HUMAN_APPROVED/REJECTED` là việc của BE, component không tự phát event.
- A11y: nhóm nút có `role="group"` + nhãn; viền trái màu `status-waiting` kèm nhãn chữ "Waiting"; mức rủi ro có chữ không chỉ màu.

#### 3.3.4 `WorkflowNode` (React Flow custom node)

```ts
export type WorkflowNodeKind = 'agent' | 'task' | 'approval' | 'condition' | 'parallel' | 'merge' | 'tool' | 'test' | 'deployment'; // 9 UI node types (OWR-2 Q-11; trước đây 8 loại theo FR-113; 'task' đồng bộ SCREEN_SPECIFICATION §6). Engine có 11 kind = 9 + start + end (WORKFLOW_ENGINE §3)
export interface WorkflowNodeData {
  id: string; kind: WorkflowNodeKind; title: string; subtitle?: string;
  assignee?: Pick<AgentSummary, 'id' | 'name' | 'avatarSrc' | 'providerId' | 'tone'>;
  tone: 'idle' | 'selected' | 'running' | 'completed' | 'failed' | 'blocked' | 'waiting-approval';
  progress?: number;                  // 0..1 khi running
  blockedReason?: string;
  attempt?: number; maxAttempts?: number;
  compact?: boolean;                  // 180x40 cho DependencyGraph
  readOnly: boolean;                  // true khi xem run; false khi chỉnh template
}
export interface WorkflowNodeProps extends NodeProps<WorkflowNodeData> {
  onRetry?: (nodeId: string) => void;
  onReview?: (nodeId: string) => void;      // mở Approval
  onOpen?: (nodeId: string) => void;
}
```
- Hình thức và bảng state: `DESIGN_SYSTEM.md` §3.21 (idle, selected, running, completed, failed, blocked, waiting-approval).
- Handle: số/hướng handle theo `kind` (condition: 1 vào, ≥ 2 ra có nhãn nhánh; parallel: 1 vào, n ra; merge: n vào, 1 ra). `readOnly` ẩn handle kéo nối.
- Events: `onRetry` (chỉ khi `failed`), `onReview` (chỉ khi `waiting-approval`), `onOpen`.
- Quan trọng: trạng thái node **chỉ** lấy từ event của run (FR-118); bản chỉnh sửa template không có tone runtime.
- A11y: node có `role="group"`, `aria-label` "Agent node Backend API, running, 60%"; Tab đi theo thứ tự topological; Enter mở panel; thao tác nối edge có thay thế bằng bàn phím (menu "Connect to…") [Đề xuất]; animation dash tắt khi reduced-motion.

#### 3.3.5 `LogViewer`

```ts
export type LogLevel = 'trace' | 'debug' | 'info' | 'warn' | 'error';
export interface LogLine {
  seq: number;                        // thứ tự tăng dần, dùng làm key và để resume
  ts: string; level: LogLevel; source: 'agent' | 'tool' | 'system' | 'sandbox';
  text: string;                       // đã redact ở BE; có thể chứa ANSI
  executionId?: string;
}
export interface LogViewerProps extends BaseProps {
  lines: readonly LogLine[];
  state?: AsyncState;
  follow: boolean; onFollowChange(v: boolean): void;
  filter?: { levels?: LogLevel[]; query?: string; sources?: LogLine['source'][] };
  onFilterChange?(f: NonNullable<LogViewerProps['filter']>): void;
  onReachTop?: () => void;            // tải thêm cũ hơn (cursor)
  maxRenderedLines?: number;          // mặc định 5000 (virtualized)
  showTimestamps?: boolean; wrap?: boolean;
  onCopy?: (selection: string) => void; onDownload?: () => void;
}
```
- Hình thức: `DESIGN_SYSTEM.md` §3.22 (CodeBlock/LogViewer: mono, virtualized, auto-follow, ANSI → màu status).
- State: ready, loading (skeleton dòng), empty ("Chưa có log"), error (Retry), disconnected (banner "Mất kết nối, đang nối lại", dựa `lastEventId`), truncated (banner "Hiển thị 5000 dòng cuối").
- Bảo mật: component **không** tự redact; nhưng không render HTML (chỉ text, ANSI parse an toàn); link được làm phẳng thành text trừ allowlist URL. Không bao giờ nhận nội dung chưa qua redaction của BE (F-04, AC-029).
- Events: `onFollowChange`, `onFilterChange`, `onReachTop`, `onCopy`, `onDownload`.
- A11y: `role="log"`, `aria-live="off"` mặc định (tránh ồn) và nút "Announce errors" bật `polite` cho dòng `error`; tìm kiếm bằng bàn phím (`/`); virtualization giữ `aria-setsize`.

#### 3.3.6 `OfficeHUD` (lớp React quanh canvas Phaser)

```ts
export interface OfficeStatusCounts { working: number; waiting: number; blocked: number; failed: number; idle: number; offline: number }
export interface OfficeHUDProps extends BaseProps {
  mode: 'live' | 'edit' | 'fullscreen';
  onModeChange(m: OfficeHUDProps['mode']): void;
  layouts: ReadonlyArray<{ id: string; name: string }>; layoutId: string; onLayoutChange(id: string): void;
  projectId?: string; projects: ReadonlyArray<{ id: string; name: string }>; onProjectChange(id?: string): void;
  counts: OfficeStatusCounts;
  statusFilter?: keyof OfficeStatusCounts; onStatusFilterChange(s?: keyof OfficeStatusCounts): void;
  followAgentId?: AgentId; onFollowChange(id?: AgentId): void;
  soundOn: boolean; onSoundChange(v: boolean): void;
  hudHidden: boolean; onHudHiddenChange(v: boolean): void;
  zoom: number; onZoomChange(z: number): void;
  ticker?: { id: string; text: string; ts: string };     // 1 event mới nhất
  fps?: number;                                           // chỉ hiển thị trong chế độ debug
  connection: 'live' | 'reconnecting' | 'offline';
}
```
- Layout: top (selector, project filter, counters, Follow, Edit, Fullscreen, Sound, Hide), bottom (minimap, zoom, ticker) — `SCREEN_SPECIFICATION.md` §3.3.
- State: live, edit, fullscreen, empty (không agent), reconnecting/offline (nhân vật mờ; không phát animation mới), error (fallback danh sách agent text; Office không phải điểm lỗi duy nhất).
- **Quy tắc nghiêm ngặt:** HUD và canvas chỉ phản ứng với event thật; không timer tạo hoạt động (FR-160, AC-091). Test harness giả lập (AC-096) nạp qua kênh riêng chỉ có trong build test.
- Events: toàn bộ qua callback ở trên; click nhân vật phát `onAgentSelect` từ `OfficeCanvasBridge` (store Zustand + event bus, D-011), không phải props của HUD.
- A11y: canvas có mô tả thay thế: danh sách agent dạng text (`AgentListFallback`) luôn tồn tại cho screen reader và khi lỗi; counters là `button` lọc, `aria-pressed`; mọi hành động HUD có phím tắt (`E` edit, `F` fullscreen, `H` hide, theo NAVIGATION_STRUCTURE §9); tôn trọng reduced-motion (tắt animation đi bộ, giữ trạng thái tĩnh).

#### 3.3.7 `AgentPanel` (panel phải của Office / Sheet của Agent)

```ts
export interface AgentPanelProps extends BaseProps {
  agent: AgentSummary & { departmentName: string; blockedReason?: string; sinceMs?: number };
  currentTask?: Pick<TaskView, 'id' | 'displayId' | 'title' | 'tone'>;
  progress?: { done: number; total: number; kind: 'subtasks' | 'tests' };
  lastLog?: string;                  // ≤ 2 dòng
  available: ReadonlyArray<'chat' | 'assign' | 'pause' | 'resume' | 'disable' | 'resolve' | 'logs' | 'workspace'>;
  onAction: (a: AgentPanelProps['available'][number], agentId: AgentId) => void;
}
```
- Hành động Pause/Disable phá huỷ nhẹ nên có `AlertDialog` (FR-029/030). Disabled actions kèm tooltip lý do.

#### 3.3.8 `ProviderStatusCard`

```ts
export type ProviderConnection = 'connected' | 'not_installed' | 'not_authenticated' | 'session_expired' | 'error' | 'unverified';
export interface ProviderStatusCardProps extends BaseProps {
  provider: { id: ProviderKind; displayName: string; version?: string; connection: ProviderConnection;
              models?: readonly string[]; activeSessions: number; lastError?: { code: string; at: string };
              usage?: { tokens: number; costUsd?: number; estimated: boolean } };
  onConnect(): void; onDisconnect(): void; onReconnect(): void; onShowGuide(kind: Exclude<ProviderConnection,'connected'>): void;
}
```
- Hiển thị hướng dẫn từng bước cho `not_installed`/`not_authenticated` (AC-023/024). **Không** có trường nhập mật khẩu hay token (FR-084); nút "Mở terminal đăng nhập" chỉ hiển thị lệnh để người dùng tự chạy. Codex ở `not_installed` hiển thị đúng trạng thái này (đã xác minh chưa cài).
- A11y: trạng thái bằng chữ + icon; vùng live thông báo khi đổi.

#### 3.3.9 `EmergencyStopButton`

```ts
export interface EmergencyStopButtonProps extends BaseProps {
  runningCount: number;
  onConfirm: () => Promise<void>;       // gọi BE; ≤ 5 s theo AC-109
  state: 'ready' | 'confirming' | 'stopping' | 'stopped' | 'error';
}
```
- Luôn hiển thị trên Topbar mọi màn (AC-112), `destructive`, mở `AlertDialog` hiển thị số agent/container bị ảnh hưởng; sau khi dừng hiển thị nút Resume (AC-110). Phím tắt chống bấm nhầm (không phím đơn).
- A11y: `aria-label` rõ; `role="alertdialog"`; kết quả thông báo `role="alert"`.

#### 3.3.10 Các domain component khác (tóm tắt hợp đồng)

| Component | Props chính | State | Events | Ghi chú |
|---|---|---|---|---|
| `DiffViewer` | `{ original: string; modified: string; language?: string; fileName: string; inline?: boolean; ownershipViolations?: readonly string[] }` | loading, ready, too-large, binary | `onOpenFile(path)` | trên Monaco; file ngoài vùng ownership được đánh dấu (AC-011) |
| `CodeBlock` | `{ code: string; language?: string; copyable?: boolean }` | — | `onCopy` | đã sanitize |
| `ProjectSwitcher` | `{ projects: ProjectRef[]; value?: ProjectId }` | closed/open/loading | `onChange` | Combobox h-40 |
| `ApprovalQueue` | `{ items: ApprovalView[]; filter; state }` | ready/loading/empty/error | `onDecide`, `onFilterChange` | gom `ApprovalCard`, badge đếm ở Sidebar |
| `TaskBoard` | `{ columns: TaskColumn[]; view: 'kanban'|'table'|'timeline'|'graph' }` | AsyncState | `onTaskOpen`, `onTaskDrop` | 4 view, dùng cùng dữ liệu |
| `DependencyGraph` | `{ nodes: WorkflowNodeData[]; edges: Edge[] }` | AsyncState | `onNodeOpen` | tái dùng `WorkflowNode compact` |
| `EventTimeline` | `{ events: DomainEventView[]; state }` | AsyncState | `onEventOpen`, `onReachEnd` | hiển thị event thật (FR-116) |
| `UsageValue` | `{ value: number; unit: 'tokens'|'usd'; estimated: boolean }` | — | — | luôn gắn nhãn "est." khi ước tính (F-19) |
| `SkillCard` | `{ skill: SkillSummary; assignedAgents: AgentSummary[] }` | default/hover/selected | `onOpen`, `onAssign` | |
| `RuleListEditor` | `{ rules: ParsedRule[]; dirty: boolean }` | editing/saving | `onChange`, `onSave`, `onDiscard` | Teach Agent: không lưu trước khi Owner xác nhận (AC-075/076) |
| `NotificationItem` | `{ n: NotificationView }` | unread/read | `onOpen`, `onDismiss` | dot `ai-accent` 6px |
| `ScreenshotMarkup` | `{ imageSrc: string; onChange(shapes: Shape[]) }` | idle/drawing | `onChange`, `onSubmit` | Stage 8 (FR-130) |

### 3.4 Bản đồ component → màn hình (rút gọn) [Đề xuất]

| Màn (SCREEN_SPECIFICATION) | Component chính |
|---|---|
| 3.2 Dashboard | `ChartCard`, `AgentChip`, `ApprovalQueue`, `EventTimeline`, `StatusBadge` |
| 3.3 Virtual Office | `OfficeHUD`, `AgentPanel`, `AgentListFallback` |
| 3.6/3.7 Agent Management/Detail | `DataTable`, `AgentAvatar`, `Tabs`, `AgentPanel` |
| 3.8 Agent Builder | `Stepper`, `FormField`, `Combobox`, `Textarea` |
| 3.11 Provider Hub | `ProviderStatusCard`, `StateBlock` |
| 3.14 Workflow Builder | `WorkflowNode`, `DependencyGraph`, `Sheet` |
| 3.15 Task Board | `TaskBoard`, `TaskCard`, `TaskRow` |
| 3.17 Approval Center | `ApprovalCard`, `ApprovalQueue`, `DiffViewer` |
| 3.18 Code Workspace | `DiffViewer`, `LogViewer`, `Resizable` |
| 3.21 Cost Management | `ChartCard`, `UsageValue`, `DataTable` |

### 3.5 Quy trình triển khai component [Đề xuất]

1. Tạo `components/app/<Name>/{Name.tsx, Name.test.tsx, Name.stories.tsx?, index.ts}`.
2. Props interface export từ cùng file; kiểu dùng chung từ `packages/contracts`.
3. Thêm vào `components/app/index.ts` chỉ khi có test pass và `axe` sạch.
4. Storybook (hoặc Ladle) là tuỳ chọn [Giả định chưa chốt]; nếu dùng, fixture lấy từ `packages/testing`.
5. Visual regression chỉ cho component ổn định (DESIGN_SYSTEM đã duyệt).

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| CS-1 | Component nhận `StatusTone` thay vì state domain thô | Ba bộ trạng thái đang lệch nhau; tách giúp UI ổn định khi domain đổi | [Đề xuất] |
| CS-2 | Component chỉ phát callback, không tự đổi trạng thái domain (kể cả drop trên TaskCard) | Trạng thái chỉ đổi qua event (FR-025/118) | [Đề xuất] |
| CS-3 | LogViewer không redact, nhưng chỉ nhận dữ liệu đã redact và render text thuần | Redaction là trách nhiệm BE (một lớp kiểm chứng được) | [Đề xuất] |
| CS-4 | OfficeHUD tách khỏi canvas Phaser, giao tiếp qua Zustand + event bus | D-011; test HUD không cần WebGL | [Đề xuất] |
| CS-5 | Luôn có `AgentListFallback` cho Office | Office không là single point of failure + a11y | [Đề xuất] |
| CS-6 | Reject/Request changes bắt buộc feedback | AC-046 | [Đề xuất] |
| CS-7 | Props interface định nghĩa ở đây là hợp đồng; đổi phải qua PR có "Contract change" | Nhiều agent làm song song | [Đề xuất] |

## 5. Dependencies

- `DESIGN_SYSTEM.md`, `DESIGN_TOKENS.md` (token CSS đã compile với Tailwind v4, AC trong DESIGN_TOKENS).
- `packages/contracts`: `AgentSummary`, `TaskView`, `ApprovalView`, `toTone()`, `ProviderKind`.
- Thư viện: shadcn/ui (Radix), TanStack Table + Virtual, React Flow, Recharts, Monaco, cmdk, Sonner, Zustand, Phaser (D-011).
- Asset avatar/sprite: chỉ CC0/MIT/OFL, có `assets-manifest.json` (D-012).
- Bảng ánh xạ trạng thái chính thức (Architect).

## 6. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Lệch giữa bộ trạng thái agent/task/tone | Hiển thị sai trạng thái | `toTone()` một nguồn + test bảng map |
| Drag-and-drop khó truy cập | Vi phạm a11y | Thay thế bàn phím, test với `axe` + thao tác thủ công |
| LogViewer chậm với log lớn | Giật UI | Virtualize, giới hạn 5000 dòng, cursor tải thêm |
| React Flow + 100+ node nặng | Workflow Builder chậm | Ẩn minimap tuỳ chọn, `onlyRenderVisibleElements` |
| Canvas Phaser khó test | Lỗi UI office lọt | Tách logic FSM/mapping ra module thuần, test Vitest |
| Asset sprite vi phạm giấy phép | Pháp lý | Manifest + kiểm CI (PROJECT_RISKS R-ASSET) |
| Props interface thay đổi giữa chừng | Vỡ nhiều màn | Quy trình Contract change |

## 7. Vấn đề chưa giải quyết

1. Bảng ánh xạ trạng thái: FR-024 (12 trạng thái agent), FR-108 (10 trạng thái task), `SCREEN_SPECIFICATION.md` §2 (idle, starting, working, waiting_input, waiting_approval, blocked, paused, failed, stopped, offline...), `StatusBadge` (8 tone). Chưa có một bảng chính thức.
2. `SCREEN_SPECIFICATION.md` §3.3 ghi canvas "PixiJS/Canvas 2D [Giả định]" trong khi D-011 chọn Phaser 3.90 qua `OfficeRenderer` (OWR-2 D-011; SCREEN_SPECIFICATION đã cập nhật). Tài liệu này theo D-011.
3. Meeting room trong Office là cơ chế thật hay animation (OI-FR-06) — ảnh hưởng `OfficeHUD`/`AgentPanel`.
4. Có dùng Storybook/Ladle không; tooling story chưa chốt.
5. `ApprovalCard`: danh sách loại approval (`kind`) cần khớp chính sách mặc định FR-199 (chờ Owner, AP-FR-04); `stage_gate` là giả định.
6. `ProviderConnection.unverified`: cần Owner xác nhận cách hiển thị Codex khi CLI chưa cài.
7. Cần quyết định có gói `packages/ui` riêng hay để trong `apps/web` (ảnh hưởng ranh giới import).
8. Độ phủ locale (EN/VI) cho component có độ dài chuỗi khác nhau: chưa kiểm tra.

## 8. Acceptance Criteria

- [ ] Mọi component trong 3.1-3.3 có file, test render từng variant/state, và `axe` không vi phạm nghiêm trọng (WCAG 2.2 AA mục tiêu).
- [ ] `toTone()` có test phủ toàn bộ trạng thái FR-024 và FR-108 (không trạng thái nào chưa map).
- [ ] `TaskCard` drop không đổi tone cục bộ; test: sau drop, tone chỉ đổi khi nhận event mô phỏng `TASK_STATE_CHANGED` (hỗ trợ AC-059, AC-060).
- [ ] `ApprovalCard` không cho gửi Reject khi `feedback` rỗng (AC-046); risk `high` yêu cầu `confirmText`.
- [ ] `LogViewer` render ≥ 50.000 dòng đã nạp (cắt còn 5000) với thao tác cuộn không rớt dưới 30 FPS trên máy tham chiếu; không render HTML thô (test XSS payload).
- [ ] `OfficeHUD` + `AgentListFallback`: khi canvas lỗi, danh sách text hiển thị và đủ hành động; không timer nào tạo hoạt động khi không có event (AC-091).
- [ ] `ProviderStatusCard` không có input nhận secret; test snapshot DOM không chứa `type="password"`.
- [ ] `EmergencyStopButton` hiện trên Topbar ở mọi route (E2E, AC-112).
- [ ] Không component nào trong `components/app` import từ `packages/testing` (lint F-03).
