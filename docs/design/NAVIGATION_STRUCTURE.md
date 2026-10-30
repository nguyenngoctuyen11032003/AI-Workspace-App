# NAVIGATION_STRUCTURE — Kiến trúc thông tin & điều hướng AI WORKSPACE

> **Tác giả:** Agent 16 (UX Research & Product Designer) · **Phase:** 0 (Documentation only) · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
> **Tài liệu liên quan:** `docs/design/DESIGN_SYSTEM.md`, `DESIGN_TOKENS.md`, `UI_UX_GUIDELINES.md`, `INTERACTION_DESIGN.md`, `RESPONSIVE_DESIGN.md` (Agent 11 — không lặp lại nội dung, chỉ tham chiếu), `SCREEN_SPECIFICATION.md`, `USER_JOURNEY.md`, `UX_RESEARCH_REPORT.md`, `USABILITY_AUDIT_CHECKLIST.md`.
>
> **Quy ước nhãn:** `[Đã xác minh]` = suy ra trực tiếp từ brief của chủ sở hữu hoặc repo tham chiếu đã đọc · `[Đề xuất]` = quyết định thiết kế của Agent 16, có thể thay đổi · `[Giả định]` = chưa có bằng chứng, cần xác nhận với agent khác · `[Cần người dùng phê duyệt]` = chủ sở hữu (Director) phải chốt trước Phase 1.

---

## 1. Mục đích

Định nghĩa **information architecture (IA)** của ứng dụng: cấu trúc sidebar, bảng route, quy tắc breadcrumb, phạm vi global vs project-scoped, deep link, danh mục lệnh Command Palette, phím tắt, phân loại notification và điểm đến của chúng, điểm vào onboarding/empty-state, và điều hướng mobile. Tài liệu này là "bản đồ" để agent frontend (Phase 2–4) dựng router, app shell và Command Palette mà không phải tự suy đoán.

## 2. Phạm vi

- **Trong phạm vi:** App shell (sidebar + topbar), route map cho 22 màn hình, quy ước query-param, breadcrumb, notification routing, keyboard shortcuts, command palette, mobile navigation.
- **Ngoài phạm vi:** Visual style của sidebar/topbar (xem `DESIGN_SYSTEM.md` của Agent 11), chi tiết từng màn hình (xem `SCREEN_SPECIFICATION.md`), auth/RBAC backend (Agent Security), API contract thực (Agent Backend). Tên entity/endpoint trong đây là **tên đề xuất** để các agent khác đối chiếu.

## 3. Nguyên tắc IA `[Đề xuất]`

1. **Status-first, project-second.** Người dùng duy nhất là Director điều hành tối đa 18 agent; câu hỏi đầu tiên mỗi lần mở app là "có gì cần tôi ngay bây giờ?" → Dashboard và Approvals phải cách tối đa 1 click từ mọi nơi.
2. **Một đối tượng, một URL.** Mọi entity (agent, task, approval, workflow, project, skill, provider) có canonical URL, copy/paste được, mở tab mới được, dùng được trong notification.
3. **Project scope tường minh.** Màn hình project-scoped luôn có `projectId` trong URL, không bao giờ ẩn ngầm trong state.
4. **Virtual Office là một *view* của cùng dữ liệu**, không phải "chế độ game" tách rời. Click vào nhân vật → cùng Agent Detail như từ Agent Management.
5. **Evidence-driven:** mọi badge/số đếm trên sidebar lấy từ dữ liệu thật (count query), không có placeholder, không có số "demo".
6. **Keyboard-first cho power user** (pattern Linear/Raycast) nhưng mọi lệnh đều có đường chuột tương đương.

## 4. App Shell

### 4.1 Sidebar (collapsible: expanded 240px / collapsed 56px icon-only) `[Đề xuất]`

| # | Group | Item | Route | Badge (nguồn dữ liệu thật) | Scope |
|---|---|---|---|---|---|
| 1 | **Overview** | Dashboard | `/dashboard` | — | Global (filter theo project tùy chọn) |
| 1 | Overview | Activity Center | `/activity` | số event chưa đọc (`notifications.unreadCount`) | Global |
| 2 | **Projects** | All Projects | `/projects` | số project `active` | Global |
| 2 | Projects | Project Detail | `/projects/:projectId` | — | Project |
| 2 | Projects | Product Preview | `/projects/:projectId/preview` | — | Project |
| 3 | **Virtual Office** | Office | `/office` | số agent `needs_attention` | Global (có thể lọc project) |
| 4 | **AI Agents** | Agents | `/agents?view=grid\|list\|org` | số agent `blocked` + `failed` | Global |
| 4 | AI Agents | Agent Builder | `/agents/new?step=1..6` | — | Global |
| 4 | AI Agents | Agent Detail | `/agents/:agentId?tab=overview\|profile\|skills\|knowledge\|provider\|tasks\|conversations\|logs\|performance\|permissions` | — | Global |
| 5 | **Organization** | Org Chart | `/org` | — | Global |
| 5 | Organization | Departments | `/org/departments` ; `/org/departments/:deptId` | — | Global |
| 6 | **Skills & Knowledge** | Skill Library | `/skills` | — | Global |
| 6 | Skills & Knowledge | Skill Editor | `/skills/:skillId/edit` ; `/skills/new` | — | Global |
| 7 | **Workflows** | Workflows | `/projects/:projectId/workflows` | số workflow run đang chạy | Project |
| 7 | Workflows | Workflow Builder | `/projects/:projectId/workflows/:workflowId?run=:runId&node=:nodeId` | — | Project |
| 8 | **Task Center** | Tasks | `/projects/:projectId/tasks?view=kanban\|table\|timeline\|graph` | số task `blocked` | Project |
| 9 | **Approvals** | Approval Center | `/approvals` ; `/approvals/:approvalId` | số approval `pending` (**ưu tiên cao nhất, màu warning**) | Global + lọc project |
| 10 | **Code Workspace** | Workspace | `/projects/:projectId/workspace?file=&line=&branch=&panel=` | — | Project |
| 11 | **QA & Security** | Reports | `/projects/:projectId/qa` ; `/projects/:projectId/qa/runs/:runId` | số finding `critical` đang mở | Project |
| 12 | **Analytics** | Cost Management | `/cost` | — | Global + lọc project/agent |
| 13 | **Provider Hub** | Providers | `/providers` ; `/providers/:providerId?tab=` | số provider `degraded\|down` | Global |
| 14 | **Settings** | Settings | `/settings/:section` (general, appearance, providers, security, notifications, office, shortcuts, about) | — | Global |

Ghi chú:
- Thứ tự 14 group giữ nguyên theo brief của chủ sở hữu `[Đã xác minh]`. Agent 16 **đề xuất** thêm divider gom thành 4 cụm để giảm tải nhận thức: *Monitor* (Overview, Virtual Office, Approvals), *Build* (Projects, Workflows, Task Center, Code Workspace, QA & Security), *Organize* (AI Agents, Organization, Skills & Knowledge), *Platform* (Analytics, Provider Hub, Settings). Việc đổi thứ tự hiển thị so với brief là `[Cần người dùng phê duyệt]`.
- Alias route ngắn (`/tasks`, `/workflows`, `/workspace`, `/qa`) redirect theo `currentProjectId`; nếu chưa có project → `/projects` với empty-state (xem §6).
- Tooltip khi collapsed: tên item + phím tắt (ví dụ `G then D`), dùng component `Tooltip` của Design System.
- Badge: component `Badge` variant `count`; màu theo severity token (`DESIGN_TOKENS.md`: `--color-status-*`). Badge pending approvals dùng `warning`, failed dùng `danger`. Debounce 1–2s để không nhấp nháy.
- **Project switcher** nằm ở đầu sidebar (dưới logo): dropdown + search + "Create project". Chọn project cập nhật `currentProjectId` (persist `localStorage`) và URL nếu đang ở màn project-scoped.
- Cuối sidebar: trạng thái realtime (WS connected / reconnecting / offline) và user menu khi collapsed.

### 4.2 Topbar (chiều cao 48px) `[Đề xuất]`

Trái → phải: **Breadcrumb** · **Project selector** (chỉ hiển thị trên màn project-scoped; mirror của switcher sidebar) · **Global search** (focus bằng `/`) · **Command palette trigger** (`Ctrl/Cmd+K`) · **Agent activity indicator** (chip `N running · M waiting · K blocked`; click → popover danh sách; click agent → `/agents/:id`) · **Provider status** (dot xanh/vàng/đỏ; click → `/providers`) · **Notifications** (bell + unread count; popover 10 mục mới nhất; "View all" → `/activity`) · **User menu** (profile, theme, shortcuts, sign out).

Trên `/office` ở chế độ fullscreen, topbar thu gọn thành HUD overlay (xem `SCREEN_SPECIFICATION.md` §3.3).

## 5. Quy tắc breadcrumb `[Đề xuất]`

1. Breadcrumb phản ánh **phân cấp dữ liệu**, không phải lịch sử click.
2. Tối đa 4 cấp; cấp giữa quá dài rút gọn thành `…` có tooltip.
3. Mẫu:
   - Global: `Agents › Dev-01` · `Skills › Code Review` · `Providers › Claude (Anthropic)`
   - Project-scoped: `Projects › Shop App › Tasks` · `Projects › Shop App › Workspace › src/app.ts` · `Projects › Shop App › QA › Run #42`
   - Tab trong Agent Detail **không** thành breadcrumb (tab là trạng thái phụ, giữ bằng `?tab=`).
4. Mục cuối không clickable; các mục trước là link. Entity rename → breadcrumb cập nhật realtime (subscribe `entity.updated`).
5. Khi vào từ Virtual Office (click nhân vật) breadcrumb vẫn là `Agents › Dev-01`, nhưng có nút ghost "← Back to Office" ở đầu nội dung để giữ mental model "văn phòng là nơi tôi đang đứng".

## 6. Project scoping

| Loại | Màn hình | Hành vi khi không có project |
|---|---|---|
| Global | Dashboard, Activity, Office, Agents, Agent Detail, Agent Builder, Org Chart, Departments, Skills, Skill Editor, Providers, Cost, Settings, Approvals (list) | Hoạt động bình thường; filter `project` tùy chọn |
| Project-scoped | Project Detail, Product Preview, Workflows, Workflow Builder, Task Center, Code Workspace, QA & Security | URL thiếu `projectId` và chưa có `currentProjectId` → redirect `/projects` kèm empty-state "Chọn hoặc tạo project" |

- Canonical: `/projects/:projectId/{tasks|workflows|workspace|qa|preview}`; alias ngắn redirect `[Đề xuất]`. Phương án thay thế (route ngắn + `?project=`) ngắn hơn nhưng ẩn project — `[Cần người dùng phê duyệt]`.
- Đổi project trong switcher khi đang ở màn project-scoped → điều hướng tới màn tương đương của project mới (giữ `view`/`tab`), không về Dashboard.
- Global screen có filter project: filter lưu trong `?project=` để share được.

## 7. Deep links `[Đề xuất]`

| Mục tiêu | URL |
|---|---|
| Agent + tab + log cụ thể | `/agents/:agentId?tab=logs&log=:logId` |
| Task trong Kanban, mở drawer | `/projects/:p/tasks?view=kanban&task=:taskId` |
| Approval chi tiết, tab diff, file | `/approvals/:approvalId?tab=diff&file=src/x.ts` |
| Node trong workflow run | `/projects/:p/workflows/:wfId?run=:runId&node=:nodeId` |
| File + dòng trong workspace | `/projects/:p/workspace?file=src/x.ts&line=120&branch=feat/x` |
| Finding trong QA report | `/projects/:p/qa/runs/:runId?finding=:findingId` |
| Nhân vật trong office (focus camera) | `/office?focus=:agentId` · `/office?project=:p&mode=edit` |
| Provider + tab usage | `/providers/:providerId?tab=usage` |
| Cost theo khoảng thời gian | `/cost?from=2026-10-01&to=2026-10-30&groupBy=agent` |
| Settings section | `/settings/notifications` |

Quy tắc: query param cho **trạng thái UI phụ** (tab, view, filter, selection); path cho **identity**. Drawer/modal mở bằng deep link đóng được bằng `Esc` và URL quay về trạng thái không có param tương ứng bằng `history.replace` (không tạo entry mới).

## 8. Command Palette (`Ctrl/Cmd+K`) `[Đề xuất]`

Pattern Raycast/Linear: một input, kết quả nhóm theo section, fuzzy search, mũi tên điều hướng, `Enter` chạy, `Ctrl+Enter` mở tab mới (với navigation). Tiền tố:

- (không tiền tố) → tìm hỗn hợp: lệnh, agent, task, project, approval, skill, file (file chỉ khi có project context).
- `>` lệnh · `@` agent · `#` task · `!` approval · `/` file trong workspace · `?` trợ giúp/shortcut.

### 8.1 Danh mục lệnh

**Navigation** (tương ứng chord `G`): Go to Dashboard / Activity / Office / Agents / Org Chart / Departments / Skills / Workflows / Tasks / Approvals / Workspace / QA / Cost / Providers / Settings; Open project…; Switch project…; Open recent (5 mục gần nhất).

**Agent actions** (cần agent context hoặc `@`): Pause agent · Resume agent · Stop agent (confirm) · Retry last task · Reassign current task… · View logs · View changes · Open workspace · Request review · Send instruction… · Edit profile · Duplicate agent · Archive agent (undo 10s).

**Task actions**: Create task… · Assign task to… · Change status… · Set priority… · Add dependency… · Open task.

**Approval actions**: Open next pending approval · Approve (chỉ khi đang ở approval detail, confirm) · Reject… · Request changes… · Jump to diff file….

**Project / Workflow**: New project from prompt… · Run workflow… · Pause workflow run · Cancel workflow run (confirm) · Open workflow builder.

**Office**: Focus agent in office · Toggle office edit mode · Follow agent camera · Mute office sounds.

**System**: Toggle theme · Toggle sidebar · Open shortcuts help · Reconnect realtime · Copy current URL · **Emergency stop all agents** (confirm bằng gõ `STOP`; xem `USER_JOURNEY.md` §J4).

### 8.2 Quy tắc an toàn
- Lệnh phá hủy (Stop, Cancel, Reject, Emergency stop, Archive, Delete) hiển thị icon danger và luôn đi qua `ConfirmDialog`. Không có undo cho stop/cancel; undo 10s cho Archive.
- Palette không "giả lập" kết quả: lệnh gọi API thật, toast hiển thị trạng thái từ response.
- Viewer role (sau này): lệnh mutation bị ẩn, không chỉ disabled.

## 9. Keyboard shortcuts `[Đề xuất]` (chi tiết motion/feedback xem `INTERACTION_DESIGN.md`)

| Phạm vi | Phím | Hành động |
|---|---|---|
| Global | `Ctrl/Cmd+K` | Command palette |
| Global | `/` | Focus global search |
| Global | `G` rồi `D / O / A / T / P / W / C / Q / S` | Go to Dashboard / Office / Agents / Tasks / aPprovals / Workflows / Code workspace / Qa / Settings |
| Global | `[` | Toggle sidebar |
| Global | `N` | Notification popover |
| Global | `?` | Shortcut help |
| Global | `Esc` | Đóng drawer/modal/popover, bỏ chọn |
| Global | `Ctrl/Cmd+Shift+.` | Emergency stop (mở confirm) |
| List/Grid (Agents, Tasks, Approvals) | `J / K`, `↑ / ↓` | Di chuyển chọn; `Enter` mở; `Space` preview drawer |
| Agents | `P` / `R` / `S` | Pause / Resume / Stop agent đang chọn (S có confirm) |
| Tasks | `C` | Tạo task; `A` assign; `1–4` đổi view Kanban/Table/Timeline/Graph |
| Approvals | `A` / `X` / `M` | Mở form Approve / Reject / Request changes (không áp dụng ngay); `]` `[` file diff tiếp/trước |
| Workflow Builder | `Delete`, `Ctrl+Z/Y`, `Ctrl+D` duplicate node, `Space+drag` pan, `F` fit view, `Ctrl+Enter` run |
| Code Workspace | Monaco mặc định; `Ctrl+P` quick open; `Ctrl+Backquote` terminal; `Ctrl+Shift+G` git panel |
| Virtual Office | `WASD` / mũi tên pan; `+ / -` zoom; `F` follow agent; `E` edit mode; `H` ẩn/hiện HUD; `M` mute |

Xung đột: khi focus trong input/textarea/Monaco/terminal, chỉ `Ctrl/Cmd+K` và `Esc` hoạt động ở mức global.

## 10. Notification taxonomy & routing `[Đề xuất]`

Nguồn: event bus backend (tên sự kiện đề xuất — Agent Backend đối chiếu). Mỗi notification có `severity` (info / success / warning / critical), `category`, `entityRef`, `deepLink`.

| Category | Sự kiện ví dụ | Severity | Kênh hiển thị | Route đích |
|---|---|---|---|---|
| `approval.requested` | Agent nộp BA doc / design / PR cần duyệt | warning | Toast persistent + badge Approvals + bell + sound (tùy chọn) | `/approvals/:id` |
| `approval.resolved` | Director đã duyệt / từ chối | info | Bell | `/approvals/:id` |
| `agent.blocked` | Cần input/permission, thiếu secret, provider lỗi | warning | Toast + badge Agents + speech bubble trong Office | `/agents/:id?tab=overview` (panel "Needs you") |
| `agent.failed` | Task lỗi sau retry, process crash | critical | Toast persistent + badge | `/agents/:id?tab=logs&log=:logId` |
| `agent.state.changed` | idle→working, finished | info | Chỉ Activity feed + Office animation (không toast) | `/agents/:id` |
| `task.status.changed` | done / blocked | info / warning | Activity feed; toast nếu `blocked` | `/projects/:p/tasks?task=:id` |
| `workflow.run.*` | started / node failed / completed / waiting approval | info / critical | Toast nếu failed hoặc completed | `/projects/:p/workflows/:wf?run=:run&node=:node` |
| `qa.report.ready` | test / security run hoàn tất | info (critical nếu có finding critical) | Toast | `/projects/:p/qa/runs/:runId` |
| `provider.status` | rate limit, auth expired, down | warning / critical | Toast + topbar dot | `/providers/:id` |
| `cost.threshold` | vượt ngân sách ngày/dự án | warning | Toast + bell | `/cost?project=:p` |
| `system` | WS mất kết nối, update available | info / warning | Banner topbar | — |

Quy tắc:
- Toast tối đa 3 cùng lúc, xếp chồng; `critical` và `approval.requested` không auto-dismiss.
- Mọi notification có mặt trong Activity Center với filter theo category/severity/agent/project; đánh dấu đã đọc khi mở deep link.
- "Attention routing" (xem `UX_RESEARCH_REPORT.md` §5): Dashboard có khu **Needs you now** gom `approval.requested`, `agent.blocked`, `agent.failed`, `provider.status critical`.
- Sound: tắt mặc định; chỉ bật được cho `approval.requested` và `agent.blocked` (học từ Pixel Agents: chime khi agent cần permission) `[Đã xác minh]` từ README Pixel Agents.

## 11. Empty-state & onboarding entry points `[Đề xuất]`

| Điều kiện | Điểm vào | Hành động chính |
|---|---|---|
| Chưa có provider kết nối | Redirect `/welcome` từ mọi route trừ `/settings`, `/providers` | "Connect Claude Code" → `/providers?connect=claude` |
| Có provider, chưa có agent | Dashboard empty-state + Office trống có ghế | "Create your first agent" → `/agents/new`; "Use template team" (tạo 5 agent mẫu: BA, Designer, Dev, QA, PM — cần confirm trước khi tạo) |
| Có agent, chưa có project | Dashboard, Tasks, Workspace, QA | "Create project from prompt" → `/projects/new` |
| Project chưa có workflow | Workflow list | "Start from template (Discovery → Design → Build → QA → Release)" |
| Không có approval pending | Approvals | "Inbox zero" + link Activity |
| Office chưa có layout | Office | "Use default layout" / "Open editor" |

Onboarding checklist (persist `onboarding.steps[]`; card ở Dashboard tới khi hoàn tất hoặc dismiss): Connect provider → Create agent → Create project → Watch office → Approve first deliverable. Mỗi bước tick chỉ khi có evidence thật (ví dụ bước "Approve" tick khi tồn tại approval `resolved`).

## 12. Mobile / tablet navigation `[Đề xuất]` — breakpoint xem `RESPONSIVE_DESIGN.md`

- **Phạm vi mobile = monitoring + approvals**, không phải authoring `[Cần người dùng phê duyệt]`.
- Bottom nav 5 mục: **Home** (Dashboard rút gọn: Needs-you + metrics) · **Agents** (list status-first) · **Approvals** · **Activity** · **More** (Office read-only, Providers, Cost, Settings).
- Màn desktop-only (hiển thị "Mở trên desktop" + nút copy link): Workflow Builder (edit), Code Workspace (edit; cho phép xem diff read-only), Skill Editor, Agent Builder (cho phép bước 1–2), Office edit mode.
- Approval trên mobile: Approve/Reject/Request changes với diff theo file, test & security summary; action bar sticky dưới.
- Tablet (≥ 768px): sidebar collapsed mặc định, bottom nav ẩn.

## 13. Quyết định & lý do

| Quyết định | Lý do | Nhãn |
|---|---|---|
| Landing mặc định = `/dashboard`, không phải Office | Dashboard trả lời "cần tôi gì" nhanh hơn; Office là view không gian/cảm xúc. Có setting "Default landing: Dashboard / Office". | `[Cần người dùng phê duyệt]` |
| Approvals là group riêng, badge warning nổi bật nhất | Nút thắt của human-in-the-loop; bỏ lỡ approval = nhiều agent đứng chờ. | `[Đề xuất]` |
| Tab Agent Detail dùng `?tab=` | 1 route / 1 entity; deep link ổn định; back button không rác. | `[Đề xuất]` |
| Canonical `/projects/:projectId/...`, alias ngắn redirect | Nhất quán, không ẩn project. | `[Cần người dùng phê duyệt]` |
| Command palette có tiền tố (`@ # ! /`) | Pattern quen thuộc (Raycast/Linear/Slack), giảm số lệnh phải nhớ. | `[Đề xuất]` |
| Emergency stop cần gõ `STOP` | Không thể hoàn tác trên 18 tiến trình; chi phí nhầm lớn hơn chi phí xác nhận. | `[Đề xuất]` |
| Mobile chỉ monitoring + approvals | Authoring không khả thi UX trên màn nhỏ; tập trung nguồn lực Phase 3–5. | `[Cần người dùng phê duyệt]` |
| 1 office toàn cục, Area theo department, filter project làm mờ agent ngoài project | Giữ 1 nơi để "nhìn cả công ty"; tránh nhiều office rời rạc như "save files" (ý tưởng Pixel Agents) ở Phase 1. | `[Đề xuất]` |

## 14. Dependencies

- Agent 11: component (`Sidebar`, `Topbar`, `Badge`, `Tooltip`, `Toast`, `ConfirmDialog`, `CommandPalette`, `Drawer`), token màu status, breakpoint.
- Agent Backend/API: entity & endpoint (`agents`, `tasks`, `approvals`, `workflows`, `workflow-runs`, `providers`, `notifications`, `projects`, `skills`, `costs`), tên event §10 — `[Giả định]` tới khi có API spec.
- Agent Realtime: WebSocket topic tương ứng category §10; debounce count.
- Agent Security: viewer role ảnh hưởng ẩn/hiện action trong palette và sidebar.

## 15. Rủi ro

- 14 group + nhiều item → quá tải nếu không gom cụm; cần test 5 người dùng (xem `UX_RESEARCH_REPORT.md` §8).
- Badge realtime trên 6–7 item với 18 agent có thể nhấp nháy liên tục → debounce và chỉ badge cho severity ≥ warning.
- Chord shortcut (`G then D`) xung đột Monaco/terminal → scope rõ (§9).
- Backend không trả `deepLink`/`entityRef` chuẩn → FE tự dựng → lệch route.
- Alias redirect theo `currentProjectId` có thể gây "nhảy" project ngoài ý muốn khi mở link cũ → hiển thị banner "Đang xem project X" trong 3s.

## 16. Vấn đề chưa giải quyết

1. Landing mặc định Dashboard hay Office? (§13)
2. Có hiển thị sidebar thành 4 cụm Monitor/Build/Organize/Platform không?
3. Viewer role có thấy Cost và Provider Hub không?
4. Có cho phép pin 2 project cùng lúc trong sidebar? (Giả định: không, Phase 1.)
5. 1 office toàn cục hay 1 office / project? (Đề xuất ở §13.)

## 17. Acceptance Criteria

- [ ] Mọi route §4 và §7 dựng trong router; mở trực tiếp bằng URL hoạt động (không phụ thuộc state trước đó).
- [ ] Sidebar hiển thị đủ 14 group; badge lấy từ API count thật; collapsed có tooltip + shortcut.
- [ ] Breadcrumb tuân §5 trên 22 màn; rename entity cập nhật breadcrumb không reload.
- [ ] Đổi project trong switcher trên màn project-scoped giữ nguyên màn hình.
- [ ] Command palette: đủ lệnh §8.1, tiền tố hoạt động, lệnh phá hủy có confirm.
- [ ] Shortcut §9 hoạt động và không kích hoạt khi focus trong input/Monaco (trừ `Ctrl+K`, `Esc`).
- [ ] Mỗi category notification §10 route đúng deep link; `critical` và `approval.requested` không auto-dismiss.
- [ ] Empty-state §11 xuất hiện đúng điều kiện, dẫn đúng route; onboarding tick chỉ khi có evidence thật.
- [ ] Mobile: bottom nav 5 mục; màn desktop-only hiển thị thông báo + copy link.
- [ ] Không có placeholder/dữ liệu giả trong shell (badge, indicator, provider dot).


> **Phụ lục sửa đổi:** xem `docs/design/HERDR_DESIGN_INTEGRATION.md` (attention state, Agent Rail, token bổ sung). Khi mâu thuẫn, phụ lục thắng cho các hạng mục đó cho đến khi chủ sở hữu duyệt (D-017..D-019).
