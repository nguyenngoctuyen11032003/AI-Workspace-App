# SCREEN_SPECIFICATION — Đặc tả 22 màn hình AI WORKSPACE

> **Tác giả:** Agent 16 (UX Research & Product Designer) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
> **Tham chiếu bắt buộc:** `NAVIGATION_STRUCTURE.md` (route, shortcut, notification), `DESIGN_SYSTEM.md` / `DESIGN_TOKENS.md` / `INTERACTION_DESIGN.md` / `RESPONSIVE_DESIGN.md` (Agent 11 — tên component dùng trong đây là tên **đề xuất** để Agent 11 đối chiếu: `PageHeader`, `MetricCard`, `DataTable`, `Card`, `Badge`, `StatusDot`, `Tabs`, `Drawer`, `Dialog`, `ConfirmDialog`, `Toast`, `EmptyState`, `Skeleton`, `ErrorState`, `FilterBar`, `SearchInput`, `SegmentedControl`, `Avatar`, `ProgressBar`, `Timeline`, `DiffViewer`, `LogViewer`, `Kbd`, `Tooltip`, `SplitPane`, `CommandPalette`).
> **Nhãn:** `[Đã xác minh]` · `[Đề xuất]` · `[Giả định]` · `[Cần người dùng phê duyệt]`.

---

## 1. Mục đích

Mô tả từng màn hình đủ chi tiết để Agent frontend dựng UI, Agent backend đối chiếu dữ liệu, QA viết test case: mục đích, mục tiêu người dùng, wireframe ASCII, vùng/component, dữ liệu và nguồn backend, hành động, trạng thái, hành vi realtime, filter, quyền, acceptance criteria. 11 màn hình lõi Phase 3 (Dashboard, Virtual Office, Agent Management, Agent Detail, Agent Builder, Workflow Builder, Task Center, Approval Center, Provider Hub, Code Workspace, QA & Security) được viết sâu hơn.

## 2. Phạm vi & quy ước chung

- **Nguồn dữ liệu:** ghi dạng `entity` / `GET /api/...` / WS `topic` — đây là tên **đề xuất** `[Giả định]` cho tới khi Agent Backend công bố API spec; mục đích là nêu rõ *màn hình cần gì*, không áp đặt contract.
- **Nguyên tắc evidence-driven `[Đã xác minh]` từ brief:** không có thanh tiến độ ước lượng, không có "fake progress". Progress luôn tính từ sự kiện có thật: task done / task total, test pass / test total, file changed, commit count, node completed / node total. Nếu không có evidence → hiển thị "Chưa có dữ liệu", không hiển thị 0% giả.
- **Trạng thái chung mọi màn:** `loading` (Skeleton đúng hình dạng nội dung, không spinner toàn trang), `empty` (EmptyState có CTA), `error` (ErrorState có Retry + mã lỗi + link log), `success` (nội dung), `offline` (banner WS mất kết nối, dữ liệu stale có timestamp).
- **Quyền:** Phase 1 chỉ có **Owner/Director** (toàn quyền). Cột "Viewer" là chuẩn bị cho tương lai (read-only, không thấy action mutation) `[Đề xuất]`.
- **Agent state machine dùng xuyên suốt** `[Đề xuất]`: `idle` · `starting` · `working` · `waiting_input` · `waiting_approval` · `blocked` · `paused` · `failed` · `stopped` · `offline`. Màu theo `DESIGN_TOKENS.md` (`--color-status-*`).
- **Phím tắt:** xem `NAVIGATION_STRUCTURE.md` §9; không lặp lại ở từng màn.

---

## 3. Đặc tả từng màn hình

### 3.1 Welcome / Onboarding — `/welcome`

**Mục đích:** Đưa chủ sở hữu từ "cài xong" đến "có provider + agent + project" trong < 10 phút, không hứa hẹn thứ chưa có.
**Mục tiêu người dùng chính:** Kết nối Claude Code (hoặc Codex) và thấy agent đầu tiên hoạt động thật.

```
+--------------------------------------------------------------------------+
|  AI WORKSPACE                                        Step 1 of 4  ○●○○   |
|                                                                          |
|   [Illustration pixel office, nhẹ, không chiếm >30% chiều cao]           |
|                                                                          |
|   Connect your first AI provider                                         |
|   ------------------------------------------------------------------     |
|   ( ) Claude Code CLI    detected: v2.x at /usr/local/bin/claude  ✓      |
|   ( ) Codex CLI          not detected  [How to install]                  |
|   ( ) Anthropic API key  [••••••••••]  [Test connection]                  |
|                                                                          |
|   Status: Testing…  →  "Connected. Model: claude-sonnet-4-5" (real ping) |
|                                                                          |
|                                    [Skip for now]   [Continue →]         |
+--------------------------------------------------------------------------+
```

- **Bước:** 1 Connect provider → 2 Create first agent (form rút gọn của Agent Builder: tên, role template, provider/model) → 3 Create project from prompt (textarea + "Discovery template") → 4 Tour (3 highlight: Office, Approvals, Emergency stop).
- **Component:** `Stepper`, `Card` chọn provider, `Input` secret, `Button`, `StatusDot`, `Toast`.
- **Dữ liệu:** `providers.detect` (`POST /api/providers/detect` — scan CLI, trả version/path), `providers.test` (`POST /api/providers/:id/test`), `onboarding` (`GET/PATCH /api/onboarding`).
- **Hành động:** Primary: Continue (disabled đến khi test connection thành công hoặc Skip). Secondary: Skip, Back, Open docs.
- **Trạng thái:** loading (đang detect, hiển thị "Scanning…" có spinner inline), empty (không detect CLI → hướng dẫn cài), error (test fail → hiển thị stderr rút gọn + Retry), success.
- **Realtime:** không cần; polling kết quả test.
- **Quyền:** Owner. Viewer không vào được.
- **AC:** [ ] Không thể hoàn tất bước 1 với provider chưa ping thành công (trừ Skip có cảnh báo). [ ] Bước 2 tạo agent thật (`POST /api/agents`) và agent xuất hiện trong Office ngay. [ ] Có thể thoát và quay lại đúng bước (persist). [ ] Không có số liệu/ảnh demo giả.

---

### 3.2 Main Dashboard — `/dashboard` (Core)

**Mục đích:** Trả lời trong 5 giây: *có gì cần tôi, công ty đang chạy thế nào, tiền đang đi đâu.*
**Mục tiêu người dùng:** Phát hiện và xử lý blocker/approval nhanh nhất; theo dõi tiến độ thật.

```
+-Sidebar-+---------------------------------------------------------------+
|         | Breadcrumb: Dashboard      [Project: All ▾] [Range: Today ▾]  |
|         +---------------------------------------------------------------+
|         | ▌NEEDS YOU NOW (3)                                            |
|         | ┌ ⚠ Approval: BA Doc "Shop App PRD" · from BA-01 · 12m  [Open]|
|         | ├ ⛔ Dev-03 blocked: missing env DATABASE_URL        [Resolve] |
|         | └ 🔴 Provider Claude: rate limited (429) · retry 2m  [View]   |
|         +---------------------------------------------------------------+
|         | [Active Projects 3][Active Agents 14/18][Running Tasks 22]     |
|         | [Completed 148][Blocked 2][Failed 1][Tokens 1.2M][Cost $38.4] |
|         +------------------------------+--------------------------------+
|         | AGENT ACTIVITY (live)        | PROJECT PROGRESS               |
|         | ● Dev-01 editing src/api.ts  | Shop App   ████████░░ 31/40    |
|         | ● QA-02 running tests (42/60)| CRM Lite   ███░░░░░░░ 6/22     |
|         | ◐ BA-01 waiting approval     | (evidence: tasks done/total)   |
|         | ○ PM-01 idle 12m             +--------------------------------+
|         | …(virtualized, 18 rows)      | WORKFLOW ACTIVITY              |
|         |                              | Build pipeline · node 4/7 · ⟳  |
|         +------------------------------+--------------------------------+
|         | PERFORMANCE ANALYTICS  [Throughput] [Success rate] [Cost/agent]|
|         | (chart; empty-state "Chưa đủ dữ liệu" nếu < 1 ngày)           |
+---------+---------------------------------------------------------------+
```

- **Vùng/Component:**
  1. `PageHeader` + filter project / time range (`Select`).
  2. **Needs You Now** (`Card` tone warning; list `AttentionItem`): gom `approval.pending`, `agent.blocked`, `agent.failed`, `provider.critical`. Sắp xếp: critical → warning → thời gian chờ dài trước. Ẩn khi rỗng (thay bằng dòng "Không có gì cần bạn lúc này ✓").
  3. **Metrics row** — 8 `MetricCard`: Active Projects, Active Agents (`working+waiting`/total), Running Tasks, Completed (trong range), Blocked, Failed, Token Usage, Estimated Cost. Mỗi card có delta so với kỳ trước **chỉ khi có dữ liệu kỳ trước**; click → màn tương ứng với filter.
  4. **Agent Activity feed** (`List` virtualized): mỗi dòng = StatusDot + tên + role + hành động hiện tại (từ tool event: editing/reading/running/testing) + thời lượng. Click → `/agents/:id`. Hover → actions nhanh Pause/View logs.
  5. **Project Progress**: `ProgressBar` với label dạng `done/total tasks`; tooltip liệt kê evidence (tasks done, tests pass, approvals resolved).
  6. **Workflow Activity**: run đang chạy, node hiện tại, `node completed/total`, link tới builder với `?run=`.
  7. **Performance Analytics**: `Tabs` 3 chart (throughput tasks/ngày, success rate %, cost per agent). Chart tuân `dataviz` guidance của Agent 11 nếu có.
- **Dữ liệu:** `GET /api/dashboard/summary?project=&from=&to=` (metrics tổng hợp) · `GET /api/attention` (needs-you) · `GET /api/agents?fields=state,currentTask,lastEvent` · `GET /api/projects?include=progress` · `GET /api/workflow-runs?status=running` · `GET /api/analytics/performance` · `GET /api/costs/summary`.
- **Realtime (WS):** `agent.state_changed`, `agent.tool_event` (cập nhật feed, throttle 500ms/agent), `task.status_changed` (metrics + progress), `approval.requested/resolved` (Needs-you), `workflow.run.*`, `provider.status`, `cost.updated` (mỗi 30s hoặc khi task kết thúc).
- **Hành động:** Primary: Open (mục Needs-you). Secondary: Resolve blocked (mở drawer agent), Pause agent, View logs, Create project, Create agent.
- **Trạng thái:** loading = Skeleton 8 card + 2 cột; empty (chưa có agent/project) = EmptyState với onboarding checklist; error một phần = card lỗi riêng lẻ (không làm hỏng cả dashboard); offline = banner + dữ liệu stale có "cập nhật lúc hh:mm".
- **Filter:** project, time range (Today / 7d / 30d / custom).
- **Quyền:** Owner: tất cả. Viewer: xem, không có action.
- **AC:** [ ] Needs-you xuất hiện ≤ 2s sau sự kiện WS. [ ] Mọi metric có tooltip nêu công thức/nguồn. [ ] Không có progress nào không có mẫu số (`x/y`). [ ] Click metric điều hướng đúng filter. [ ] Feed không giật khi 18 agent cập nhật liên tục (throttle). [ ] Dashboard load < 1.5s với 18 agent/500 task (`[Giả định]` ngưỡng, chốt với Agent Performance).

---

### 3.3 Virtual Office — `/office` (Core; vùng duy nhất có phong cách game) `[Đã xác minh]`

**Mục đích:** Cho Director "nhìn thấy công ty" bằng một cái liếc: ai đang làm, ai đang chờ, ai bị kẹt — và vào được chi tiết trong 1 click. Office **không** thay thế các màn hình chuyên nghiệp; nó là lớp nhận thức ngoại vi (ambient awareness).
**Mục tiêu người dùng:** Phát hiện agent cần giúp (speech bubble), nhảy đến đúng chỗ; thỉnh thoảng sắp xếp lại văn phòng theo department.

**Bài học từ tham chiếu `[Đã xác minh]`:** Pixel Agents — 1 agent = 1 nhân vật; animation theo hành động thật (type khi edit, read khi search, bubble khi chờ input/permission); Areas map thư mục → chỗ ngồi; layout editor; chime tùy chọn. my-virtual-office — dashboard panel bên phải (branch/department, agent directory với status, activity log), chat khi click agent, editor kéo thả, branch màu theo tường; tuy nhiên có nhiều yếu tố trang trí (pet, TV, thời tiết, meeting giả lập) tạo "hoạt động giả" — **không đưa vào** vì vi phạm evidence-driven.

```
+-Sidebar-+---------------------------------------------------------------+
|(collapse| HUD top: [Office ▾ Default] [Project: All ▾] ● 14 working ◐3 ⛔1 |
|  auto)  |          [Follow: —] [Edit mode] [⛶ Fullscreen] [🔊] [H hide] |
|         +--------------------------------------------+------------------+
|         |  ┌ ENGINEERING ─────────┐ ┌ PRODUCT ──────┐ | AGENT PANEL      |
|         |  │ [Dev-01]💻 [Dev-02]💻 │ │ [BA-01] 💬?   │ | (docked right,   |
|         |  │ [Dev-03]⛔  [QA-01]🧪 │ │ [PM-01] zz    │ |  collapsible)    |
|         |  └──────────────────────┘ └───────────────┘ | ▌Dev-03 · Backend|
|         |  ┌ DESIGN ──────────────┐   [Meeting room]  | ⛔ blocked 4m    |
|         |  │ [UX-01]🎨 [UX-02]💻  │   (chỉ hiển thị  | Task: Auth API   |
|         |  └──────────────────────┘    khi có review | Provider: Claude |
|         |   Floor plan pixel art 32px tiles           | Model: sonnet-4.5|
|         |   bubble "?" = waiting_input                | Progress 3/7 ✓   |
|         |   bubble "✋" = waiting_approval            | Last log: "env   |
|         |   bubble "⛔" = blocked/failed              |  DATABASE_URL…"  |
|         |                                            | [Resolve][Logs]  |
|         |                                            | [Pause][Stop]    |
|         |                                            | [Open workspace] |
|         +--------------------------------------------+------------------+
|         | HUD bottom: minimap · zoom 100% · "Dev-01 edited 3 files · 2s"|
+---------+---------------------------------------------------------------+
```

**Ba chế độ:** `Live` (mặc định) · `Edit` (layout editor) · `Fullscreen` (ẩn sidebar + topbar, HUD overlay).

**Vùng/Component:**
1. **Canvas** (PixiJS/Canvas 2D — chọn công nghệ thuộc Agent Frontend `[Giả định]`): tiles, tường, bàn, nhân vật. Mỗi department = một **Area** có màu viền và nhãn; agent ngồi trong Area của department mình (`agent.departmentId`). Agent ngoài project đang lọc → mờ 40%.
2. **Nhân vật** = `agent`; animation map trực tiếp từ state/tool event: `working` + tool `edit/write` → gõ phím; `read/search` → đọc tài liệu; `bash/test` → terminal; `idle` → ngồi/đi lại nhẹ (**không** đi uống cà phê, xem TV); `waiting_input` → bubble "?"; `waiting_approval` → bubble "✋" + đi tới "Review desk"; `blocked/failed` → bubble "⛔" đỏ, nhân vật đứng; `paused` → bubble "⏸"; `offline/stopped` → ghế trống + nameplate xám. Review/handoff giữa 2 agent thật (task handoff event) → hai nhân vật tới meeting table; **không** có meeting giả lập.
3. **HUD top:** office selector (layout), project filter, bộ đếm trạng thái (click → lọc highlight nhân vật), Follow (camera bám agent), Edit mode, Fullscreen, Sound, Hide HUD.
4. **Agent Panel** (docked phải, `Drawer` inline, 320px): tên/role/department/provider/model/state + thời lượng; task hiện tại (link); progress evidence (`subtasks done/total` hoặc `tests pass/total`); last log line (monospace, 2 dòng); actions: Resolve (nếu blocked: mở form cung cấp input/secret), Pause/Resume, Stop (confirm), Retry, Reassign, View logs, View changes, Open workspace, Request review, Send instruction (textarea). Panel hiển thị khi click nhân vật hoặc `?focus=`.
5. **HUD bottom:** minimap, zoom, ticker 1 dòng event mới nhất (không phải log đầy đủ).
6. **Layout Editor mode** (`E`): toolbar trái (Select / Floor / Wall / Furniture / Area / Label / Eraser), palette tài sản (desk, chair, meeting table, whiteboard, plant, server rack, review desk…), thuộc tính phải (Area: tên, màu, department mapping, số chỗ), undo/redo, Save / Save as / Reset to default, Import/Export JSON `[Đề xuất]` (học từ Pixel Agents/MVO nhưng định dạng riêng — không dùng `.mvo-layout.json` vì AGPL, xem `docs/research/MY_VIRTUAL_OFFICE_ANALYSIS.md`). Trong edit mode, nhân vật đứng yên và mờ để không gây rối.

- **Dữ liệu:** `GET /api/office/layouts`, `GET/PUT /api/office/layouts/:id` (tiles, walls, furniture, areas[] với `departmentId`, seats[]) · `GET /api/agents?include=state,currentTask,seat` · `GET /api/departments` · WS `agent.state_changed`, `agent.tool_event` (loại tool để chọn animation), `agent.seat_assigned`, `task.handoff`, `approval.requested`.
- **Hành động:** Primary: click nhân vật → Agent Panel; double-click → `/agents/:id`. Secondary: lọc theo trạng thái; Follow; Edit layout; Fullscreen; toggle sound; right-click nhân vật → context menu (cùng action panel).
- **Trạng thái:** loading = sơ đồ sàn skeleton + "Đang tải văn phòng"; empty (chưa agent) = sàn trống + ghế có nhãn "Create your first agent"; error = fallback sang danh sách agent (text) với nút Retry (Office không được là single point of failure); offline = nhân vật đóng băng + banner "Mất kết nối — trạng thái tại hh:mm".
- **Filter:** project, department, state (chip ở HUD).
- **Hiệu năng `[Đề xuất]`:** 18 nhân vật + 60 FPS mục tiêu trên laptop tích hợp; giảm xuống 30 FPS khi tab ẩn; tắt animation khi `prefers-reduced-motion`.
- **Quyền:** Owner: tất cả. Viewer: xem, panel read-only, không Edit.
- **AC:** [ ] Mỗi trạng thái agent trong state machine có biểu diễn hình ảnh khác biệt và có legend (`?` icon ở HUD). [ ] Bubble xuất hiện ≤ 1s sau `agent.blocked`/`waiting_*`. [ ] Không có animation nào không gắn với event thật (kiểm bằng log: mọi animation transition có event id). [ ] Click nhân vật → panel đầy đủ 8 trường + 10 action. [ ] Edit mode lưu layout và reload giữ nguyên; Reset to default có confirm. [ ] Office lỗi render → fallback list, app vẫn dùng được. [ ] 18 agent, 60 FPS trên máy tham chiếu (chốt với Agent Performance). [ ] Reduced-motion tắt animation đi lại.

---

### 3.4 Organization Chart — `/org`

**Mục đích:** Nhìn cấu trúc công ty ảo (Director → Departments → Agents), kéo thả để đổi department/manager.
**Mục tiêu:** Hiểu ai báo cáo ai; tái cấu trúc nhanh khi thêm agent.

```
+---------------------------------------------------------------+
| Org Chart        [Layout: Tree ▾] [Dept ▾] [Fit] [+ Department]|
|                      ┌─────────────┐                           |
|                      │  Director   │ (you)                     |
|                      └──────┬──────┘                           |
|        ┌────────────────────┼────────────────────┐             |
|   ┌────┴────┐          ┌────┴────┐          ┌────┴────┐        |
|   │ Product │ 3        │  Eng    │ 8        │  QA     │ 3      |
|   └────┬────┘          └────┬────┘          └────┬────┘        |
|  [BA-01 ●][PM-01 ○]  [Dev-01 ●][Dev-02 ●]…  [QA-01 🧪]…         |
+---------------------------------------------------------------+
```

- **Component:** React Flow (read-mostly) hoặc tree layout; `AgentNode` (Avatar, tên, role, StatusDot), `DepartmentNode` (tên, số agent, màu Area), `Minimap`.
- **Dữ liệu:** `GET /api/org/tree` (departments[], agents[] với `departmentId`, `managerAgentId`), WS `agent.state_changed` (dot), `org.updated`.
- **Hành động:** Primary: kéo agent sang department khác (ConfirmDialog: "Chuyển Dev-02 sang QA? Seat trong Office sẽ đổi"). Secondary: Add department, Collapse/expand nhánh, Export PNG, click node → `/agents/:id` hoặc `/org/departments/:id`.
- **Trạng thái:** empty = "Chưa có department — tạo hoặc dùng mẫu (Product, Design, Engineering, QA, DevOps)"; error chuẩn.
- **Realtime:** StatusDot; cấu trúc cập nhật khi `org.updated`.
- **Quyền:** Owner kéo thả; Viewer chỉ xem.
- **AC:** [ ] Kéo thả có confirm và đồng bộ với Office Area ≤ 2s. [ ] 18 agent vừa màn 1440px không scroll ngang khi Fit. [ ] Dot trạng thái đúng với Agent Management.

---

### 3.5 Department Management — `/org/departments`, `/org/departments/:deptId`

**Mục đích:** CRUD department: tên, mô tả, màu, lead agent, mapping Office Area, ngân sách token/ngày `[Đề xuất]`.

```
+-----------------------------+---------------------------------+
| Departments  [+ New]        | Engineering                     |
| ● Product        3 agents   | Color ■  Lead: Dev-01 ▾         |
| ● Engineering    8 agents   | Office area: Area "ENG" ▾       |
| ● QA             3 agents   | Daily token budget: 2,000,000   |
| ● Design         2 agents   | Agents (8)  [Add agent ▾]       |
|                             |  Dev-01 Backend  ● working  [×] |
|                             |  …                              |
|                             | Skills default for dept (chips) |
|                             | [Save]  [Archive department]    |
+-----------------------------+---------------------------------+
```

- **Component:** `SplitPane` (list 280px + detail), `Form`, `ColorPicker` (token palette), `Select`, `DataTable` nhỏ, `ConfirmDialog`.
- **Dữ liệu:** `departments` (`GET/POST/PATCH/DELETE /api/departments`), `agents?departmentId=`, `office.areas`, `costs/summary?groupBy=department`.
- **Hành động:** Primary: Save. Secondary: Add/remove agent, Archive (chỉ khi 0 agent hoặc chọn department đích), Set lead.
- **Trạng thái:** empty = CTA "Create from template"; error form inline per-field.
- **Realtime:** số agent & state trong list.
- **AC:** [ ] Không archive được department còn agent nếu chưa chọn đích. [ ] Budget vượt → cảnh báo trên Cost và Dashboard (`cost.threshold`). [ ] Mapping Area phản ánh ngay trong Office.

---

### 3.6 Agent Management — `/agents?view=grid|list|org` (Core)

**Mục đích:** Màn điều khiển 18 agent song song: thấy trạng thái, tiến độ evidence, hành động hàng loạt.
**Mục tiêu:** "Ai đang kẹt? Ai rảnh? Dừng/giao lại việc trong 2 click."

```
+---------------------------------------------------------------------------+
| Agents (18)   [Grid|List|Org]  [Search @]  [Dept ▾][State ▾][Provider ▾]  |
|               Sort: Needs attention ▾        [⏸ Pause all][⛔ Stop all]    |
|  ⚠ 2 need attention: Dev-03 blocked · BA-01 waiting approval   [Show only]|
+---------------------------------------------------------------------------+
| GRID (card 300px, 3–4 cột)                                                |
| ┌────────────────────────────┐ ┌────────────────────────────┐             |
| │ [Av] Dev-03 · Backend Dev  │ │ [Av] QA-02 · QA Engineer   │             |
| │ Engineering · Claude/sonnet│ │ QA · Codex/gpt-5-codex     │             |
| │ ⛔ BLOCKED 4m              │ │ ● WORKING 23m              │             |
| │ Task: Auth API (#142)      │ │ Task: E2E checkout (#150)  │             |
| │ Evidence: 3/7 subtasks ✓   │ │ Evidence: tests 42/60 pass │             |
| │ > missing env DATABASE_URL │ │ > running playwright…      │             |
| │ [Resolve][Logs][⋯]         │ │ [Pause][Logs][⋯]           │             |
| └────────────────────────────┘ └────────────────────────────┘             |
+---------------------------------------------------------------------------+
| LIST (DataTable): ☐ | Status | Name | Role | Dept | Provider/Model |       |
|   Current task | Progress (x/y) | Runtime | Last log | Cost today | Actions|
+---------------------------------------------------------------------------+
```

- **Vùng/Component:** `PageHeader` + `SegmentedControl` view; `FilterBar`; **Attention banner** (chỉ hiện khi có agent `blocked|failed|waiting_*`; "Show only" lọc nhanh); `AgentCard` (grid) / `DataTable` (list, virtualized, cột ẩn/hiện, sort, bulk select) / Org view = embed Org Chart §3.4 rút gọn. `⋯` menu = đủ 10 action: Pause, Resume, Stop, Retry, Reassign, View logs, View changes, Open workspace, Request review, Edit.
- **Bulk actions (list):** Pause selected, Resume selected, Stop selected (ConfirmDialog liệt kê tên), Move to department, Set provider/model (chỉ khi idle).
- **Sort mặc định `[Đề xuất]`:** Needs attention (blocked/failed/waiting) → working → idle → offline; sau đó theo thời gian chờ giảm dần.
- **Dữ liệu:** `GET /api/agents?include=state,currentTask,progress,lastLog,provider,costToday` · `POST /api/agents/:id/{pause|resume|stop|retry|reassign}` · `POST /api/agents/bulk` · WS `agent.state_changed`, `agent.tool_event` (lastLog, throttle), `task.progress`.
- **Trạng thái:** loading = skeleton card; empty = CTA Create agent / Use template team; error chuẩn; partial (một agent fetch lỗi) → card lỗi riêng.
- **Filter/search:** department, state, provider, project (tasks đang giao), role; search tên/role; filter lưu vào URL.
- **Quyền:** Owner. Viewer: ẩn action, ẩn bulk.
- **AC:** [ ] Card hiển thị đủ: name, role, provider, model, current task, state, evidence progress, runtime, last log. [ ] Stop/Stop all có confirm liệt kê tên; không có undo, cảnh báo rõ. [ ] Đổi view giữ filter. [ ] 18 card cập nhật realtime không layout shift (giữ chiều cao cố định). [ ] "Needs attention" luôn ở đầu. [ ] Reassign mở dialog chọn agent đích có gợi ý "idle + cùng department + có skill yêu cầu".

---

### 3.7 Agent Detail — `/agents/:agentId?tab=` (Core)

**Mục đích:** Hồ sơ + bảng điều khiển của một agent: cấu hình, năng lực, lịch sử, hiệu năng, quyền.
**Mục tiêu:** Chẩn đoán vì sao agent kẹt/chậm; điều chỉnh skill/provider/permission; xem log.

```
+---------------------------------------------------------------------------+
| ← Back to Office  Agents › Dev-03                                         |
| [Av] Dev-03  Backend Developer · Engineering · Claude / claude-sonnet-4-5 |
| ⛔ BLOCKED 4m · Task #142 Auth API · 3/7 ✓                                 |
| [Resolve] [Pause] [Stop] [Retry] [Reassign] [Open workspace] [Request review]
+---------------------------------------------------------------------------+
| Overview | Profile | Skills | Knowledge | Provider | Tasks | Conversations |
| Activity Logs | Performance | Permissions                                 |
+---------------------------------------------------------------------------+
| OVERVIEW                                                                  |
| ┌ Needs you ───────────────────────────┐ ┌ Today ─────────────┐          |
| │ Missing env DATABASE_URL             │ │ Tasks 2 done 1 blk │          |
| │ [Provide value ▸] [Mark as resolved] │ │ Tokens 184k $5.2   │          |
| └──────────────────────────────────────┘ │ Runtime 3h12m      │          |
| Current task card · Recent changes (5 files) · Last 10 log lines          |
+---------------------------------------------------------------------------+
```

- **Header** cố định: Avatar, tên (inline-edit), role, department, provider/model (`Badge`), state + thời lượng, task hiện tại, evidence progress; action bar (ẩn action không hợp lệ theo state — ví dụ Resume chỉ khi paused).
- **Tabs** (`?tab=`):
  - **Overview:** Needs-you card (nếu có), KPI hôm nay (tasks, tokens, cost, runtime), current task, recent changes (link View changes), last logs.
  - **Profile:** tên, avatar/sprite (chọn trong bộ ký tự pixel — `[Đề xuất]` 8 bộ ký tự tự vẽ, không dùng asset của repo tham chiếu vì license), role, department, mô tả, system prompt (Monaco, versioned, diff khi sửa), persona tone, working hours/concurrency limit.
  - **Skills:** skill đã gắn (chip + version), toggle, "Teach skill" → chọn từ Library hoặc tạo mới (xem `USER_JOURNEY.md` §J5); bảng usage count per skill (evidence).
  - **Knowledge:** nguồn tri thức (repo paths, docs, URLs, files), trạng thái index (chunks, last indexed), Re-index.
  - **Provider:** provider, model, temperature/max tokens, permission mode (ask / auto-edit / full — `[Cần người dùng phê duyệt]` cho phép full), fallback provider, test connection.
  - **Tasks:** DataTable task của agent (status, project, started, duration, result), filter.
  - **Conversations:** phiên hội thoại với Director và với agent khác (handoff); mỗi phiên = transcript (markdown render, tool call collapsed); ô gửi instruction mới.
  - **Activity Logs:** `LogViewer` streaming (level filter, search, pause auto-scroll, copy, download); mỗi dòng có timestamp, tool, duration; deep link `?log=`.
  - **Performance:** success rate, avg task duration, tokens/task, cost/task, retries, review rejections — chart theo tuần; empty nếu < 5 task.
  - **Permissions:** file-system scope (allow/deny paths), network, shell commands allowlist, tool allowlist (MCP), budget/ngày, "require approval for: write to main / deploy / delete".
- **Dữ liệu:** `GET /api/agents/:id` · `/api/agents/:id/{tasks|conversations|logs|metrics|skills|knowledge|permissions}` · `PATCH /api/agents/:id` · WS `agent.*`, `agent.log` (stream).
- **Trạng thái:** tab loading riêng; Logs empty = "Agent chưa chạy lần nào"; Performance empty = "Cần ≥ 5 task hoàn tất".
- **Quyền:** Owner sửa; Viewer read-only, không thấy Permissions/Provider secret.
- **AC:** [ ] 10 tab đúng tên, deep link `?tab=` hoạt động. [ ] Action bar chỉ hiện action hợp lệ với state. [ ] Sửa system prompt tạo version mới, có diff và rollback. [ ] Log stream không mất dòng khi 100 dòng/giây (buffer + virtualized). [ ] Thay provider khi agent working bị chặn với giải thích. [ ] Permissions lưu thành công mới áp dụng cho task kế tiếp (hiển thị "áp dụng từ task tiếp theo").

---

### 3.8 Agent Builder (wizard) — `/agents/new?step=1..6` (Core)

**Mục đích:** Tạo agent chất lượng trong < 3 phút với mặc định tốt; cho phép đi sâu khi cần.
**Mục tiêu:** Có agent sẵn sàng làm việc, đã test được "Hello" với provider thật.

```
+---------------------------------------------------------------------------+
| New Agent          ①Role ②Identity ③Provider ④Skills ⑤Permissions ⑥Review |
+------------------------------------------+--------------------------------+
| STEP 1 · Choose a role template          | PREVIEW (live)                 |
| [Business Analyst][Product Manager]      | [sprite] Dev-07                |
| [UX Designer][Frontend Dev][Backend Dev] | Backend Developer · Eng        |
| [QA Engineer][Security][DevOps][Custom]  | Claude · claude-sonnet-4-5     |
|                                          | Skills: Code review, Tests     |
| Template sets: system prompt, default    | Permissions: ask before write  |
| skills, permissions. You can edit later. | Est. cost/hour: — (no data yet)|
|                       [Back] [Continue]  |                                |
+------------------------------------------+--------------------------------+
```

- **Bước:** 1 Role template → 2 Identity (tên, sprite, department, mô tả, tone) → 3 Provider & model (chỉ provider `connected`; model list thật từ provider; **Test prompt** gửi 1 request thật, hiển thị response + latency + token) → 4 Skills & Knowledge (chọn từ library, thêm nguồn tri thức) → 5 Permissions & budget → 6 Review & Create (summary + "Create & seat in Office").
- **Component:** `Stepper`, `TemplateCard`, `Form` + validation inline, `Select` searchable, `Monaco` (system prompt), `Preview` pane cố định phải, `Toast`.
- **Dữ liệu:** `GET /api/agent-templates` · `GET /api/providers?status=connected` · `GET /api/providers/:id/models` · `POST /api/providers/:id/test-prompt` · `GET /api/skills` · `POST /api/agents` · `POST /api/agents/:id/seat` (hoặc tự động).
- **Hành động:** Continue / Back / Save draft (lưu `localStorage` + server draft) / Create. Secondary: Duplicate from existing agent.
- **Trạng thái:** provider chưa kết nối → bước 3 hiển thị EmptyState "Connect provider" (link Provider Hub, quay lại giữ draft); test prompt lỗi → hiển thị lỗi thật, không cho Continue nếu chưa test thành công (có "Skip test" với cảnh báo `[Đề xuất]`); đã đạt 18 agent → header cảnh báo "Giới hạn 18 agent đồng thời" nhưng vẫn cho tạo ở trạng thái offline `[Cần người dùng phê duyệt]`.
- **Quyền:** Owner.
- **AC:** [ ] Hoàn tất 6 bước với template < 3 phút (đo trong usability test). [ ] Preview cập nhật theo mỗi field. [ ] Không có "Est. cost" khi chưa có dữ liệu (hiển thị "—"). [ ] Agent tạo xong xuất hiện trong Office và Agents ≤ 2s. [ ] Thoát giữa chừng → quay lại đúng bước.

---

### 3.9 Skill Library — `/skills`

**Mục đích:** Kho kỹ năng tái sử dụng (prompt + tools + checklist + examples) gắn cho agent.

```
+---------------------------------------------------------------------------+
| Skills (24)  [Search] [Category ▾][Source: Built-in|Custom|Imported ▾]  [+ New]
| ┌ Code Review ──────────┐ ┌ Write PRD ───────────┐ ┌ E2E Testing ────────┐
| │ v3 · Engineering      │ │ v1 · Product         │ │ v2 · QA             │
| │ Used by 6 agents      │ │ Used by 2 agents     │ │ Used by 3 agents    │
| │ Success 91% (34 runs) │ │ — (no runs yet)      │ │ 78% (9 runs)        │
| │ [Open][Assign ▾]      │ │ [Open][Assign ▾]     │ │ [Open][Assign ▾]    │
+---------------------------------------------------------------------------+
```

- **Component:** `FilterBar`, `SkillCard`, `Drawer` preview, `Dialog` Assign (chọn nhiều agent).
- **Dữ liệu:** `GET /api/skills?include=usage,metrics` · `POST /api/skills/:id/assign` · `POST /api/skills/import`.
- **Hành động:** New, Open (editor), Assign to agents, Duplicate, Import/Export (JSON/Markdown `SKILL.md` style `[Đề xuất]`), Archive.
- **Trạng thái:** empty = "Bắt đầu từ bộ built-in" (liệt kê các skill mặc định sẽ được tạo thật).
- **AC:** [ ] Thống kê success/usage là dữ liệu thật, "—" khi chưa có. [ ] Assign hàng loạt cập nhật Agent Detail › Skills ngay.

---

### 3.10 Skill Editor — `/skills/:skillId/edit`, `/skills/new`

```
+-----------------------------+---------------------------------------------+
| Metadata                    | Instructions (Markdown/Monaco)              |
| Name, Category, Version     | # Code Review                               |
| Tools required (chips)      | ## When to use …                            |
| Inputs schema (JSON)        | ## Steps …                                  |
| Checklist (acceptance)      +---------------------------------------------+
| Examples (few-shot)         | Test run: [Agent ▾] [Sample input] [Run ▶]  |
| [Save v4] [Publish] [History]| Output (real) · tokens · latency           |
+-----------------------------+---------------------------------------------+
```

- **Dữ liệu:** `GET/PUT /api/skills/:id`, `GET /api/skills/:id/versions`, `POST /api/skills/:id/test-run` (chạy thật qua agent chọn).
- **Hành động:** Save (tạo version), Publish (áp dụng cho agent đang gắn, có confirm liệt kê agent), Test run, Diff với version trước, Rollback.
- **Trạng thái:** unsaved-changes guard khi rời trang; test run error hiển thị log thật.
- **AC:** [ ] Mỗi Save tạo version, diff xem được. [ ] Publish có confirm liệt kê agent ảnh hưởng. [ ] Test run là request thật, hiển thị token/latency.

---

### 3.11 AI Provider Hub — `/providers`, `/providers/:providerId?tab=` (Core)

**Mục đích:** Kết nối, giám sát sức khỏe, hạn mức, chi phí và phân bổ provider/model cho agent.
**Mục tiêu:** Biết ngay provider nào lỗi, agent nào bị ảnh hưởng, chuyển fallback.

```
+---------------------------------------------------------------------------+
| Providers              [+ Connect provider ▾ Claude Code · Codex · API key]|
| ┌ Claude Code (CLI) ─────────┐ ┌ Codex (CLI) ───────────┐ ┌ Anthropic API ┐
| │ ● Healthy · v2.1.0         │ │ ⚠ Rate limited (429)   │ │ ○ Not set up  │
| │ 11 agents · 842k tok today │ │ 5 agents · retry in 2m │ │ [Connect]     │
| │ p50 latency 1.8s           │ │ 2 tasks waiting        │ │               │
| │ [Manage]                   │ │ [Manage][Switch affected…]│               │
| └────────────────────────────┘ └────────────────────────┘ └───────────────┘
+---------------------------------------------------------------------------+
| Provider detail · Tabs: Overview | Models | Agents | Usage | Limits | Settings | Logs
+---------------------------------------------------------------------------+
```

- **Vùng:** card per provider với StatusDot (healthy / degraded / rate_limited / auth_error / down / not_configured), số agent dùng, token hôm nay, latency p50 (đo thật từ request), tác động (task đang chờ).
- **Detail tabs:** Overview (health timeline 24h), Models (danh sách model thật từ provider, context window, pricing nếu provider cung cấp, đặt default), Agents (agent dùng provider; bulk switch sang provider khác — chỉ agent idle, còn lại áp dụng cho task tiếp theo), Usage (tokens/cost theo giờ/ngày/agent), Limits (RPM/TPM/budget ngày; cảnh báo ngưỡng), Settings (binary path, config dir, API key — secret masked, test connection), Logs (request log rút gọn: time, agent, model, tokens, latency, status).
- **Dữ liệu:** `GET /api/providers?include=health,usage` · `POST /api/providers` · `POST /api/providers/:id/test` · `GET /api/providers/:id/models` · `GET /api/providers/:id/usage` · `PATCH /api/providers/:id/limits` · WS `provider.status`, `provider.usage`.
- **Hành động:** Connect, Test, Set default model, Switch affected agents, Pause provider (chặn task mới), Rotate key, Disconnect (confirm nếu còn agent).
- **Trạng thái:** not_configured card là empty-state nội tuyến; auth_error → hướng dẫn login CLI (lệnh hiển thị, không chạy tự động) `[Đề xuất]`.
- **Quyền:** Owner; Viewer không thấy Settings/keys.
- **AC:** [ ] Health thay đổi → card + topbar dot ≤ 2s. [ ] "Switch affected agents" liệt kê agent và kết quả từng agent. [ ] Secret không bao giờ hiển thị plain; có "Reveal" với confirm `[Cần người dùng phê duyệt]`. [ ] Usage khớp Cost Management (cùng nguồn).

---

### 3.12 Project Management — `/projects`

```
+---------------------------------------------------------------------------+
| Projects (3)   [Search] [Status ▾] [Grid|Table]        [+ New from prompt] |
| ┌ Shop App ──────────────────────────┐ ┌ CRM Lite ─────────────────────┐   |
| │ ● Building · 31/40 tasks · 6 agents│ │ ◐ Awaiting approval (Design)  │   |
| │ Workflow: Build pipeline node 4/7  │ │ 6/22 tasks · 3 agents         │   |
| │ Cost $22.1 · Tests 118/120 pass    │ │ Cost $7.8 · Tests —           │   |
| │ ⚠ 1 blocked · 1 approval pending   │ │ [Open][Office][Approvals]     │   |
+---------------------------------------------------------------------------+
```

- **Dữ liệu:** `GET /api/projects?include=progress,agents,cost,tests,attention` · `POST /api/projects` (from prompt: `{prompt, template}` → tạo project + workflow từ template + task Discovery đầu tiên).
- **Hành động:** New from prompt (Dialog: textarea prompt, chọn template workflow, chọn team/department, ngân sách), Open, Archive, Clone settings.
- **Trạng thái:** empty = CTA New from prompt (có gợi ý prompt mẫu, đánh dấu rõ là gợi ý).
- **AC:** [ ] Card chỉ hiển thị số liệu có evidence. [ ] Tạo từ prompt → điều hướng Project Detail với task Discovery đang chạy thật.

---

### 3.13 Project Detail — `/projects/:projectId`

```
+---------------------------------------------------------------------------+
| Projects › Shop App       ● Building        [Pause project][⋯]            |
| Tabs: Overview | Team | Workflow | Tasks | Approvals | Deliverables | Cost | Settings
+---------------------------------------------------------------------------+
| Overview: Phase timeline (Discovery ✓ → Design ✓ → Build ● → QA → Release)|
|  Attention (blocked/approvals) · Progress evidence · Recent deliverables   |
|  (PRD v2 ✓ approved, Design spec v1 ✓, PR #12 awaiting) · Activity feed    |
+---------------------------------------------------------------------------+
```

- Tabs nhúng: Team (agent trong project + load), Workflow (link builder với run hiện tại), Tasks (Task Center nhúng), Approvals (lọc project), Deliverables (artifact: docs, designs, PRs, builds, reports; mỗi artifact có version + approval status), Cost (lọc project), Settings (repo path/URL, branch chính, env secrets ref, phase gates: yêu cầu approval ở phase nào).
- **Dữ liệu:** `GET /api/projects/:id?include=phases,attention,deliverables` · WS `project.*`, `task.*`, `approval.*`.
- **AC:** [ ] Phase chỉ tick ✓ khi gate approval đã resolved. [ ] Deliverables là artifact thật có link mở được.

---

### 3.14 Workflow Builder — `/projects/:p/workflows/:workflowId?run=&node=` (Core, React Flow) `[Đã xác minh]`

**Mục đích:** Thiết kế và theo dõi pipeline công việc giữa agent: ai làm gì, khi nào cần duyệt, rẽ nhánh, chạy song song, test, deploy.
**Mục tiêu:** Dựng pipeline từ template trong 5 phút; khi chạy, thấy node nào đang chạy/kẹt và nhảy vào evidence.

```
+---------------------------------------------------------------------------+
| Projects › Shop App › Workflows › Build pipeline   [Draft|Published v3]    |
| [▶ Run][⏸][■ Cancel] Run #17 · node 4/7 · 12m   [Validate][Save][History] |
+---------+-------------------------------------------------+---------------+
| PALETTE |  CANVAS (React Flow)                            | INSPECTOR     |
| Agent   |   [Start]→[Agent BA: Write PRD]→[Approval: PRD] | Node: Approval|
| Task    |        ↓                                        | Title         |
| Approval|   [Condition: approved?]─no→[Agent BA: Revise]  | Approvers: you|
| Condition|       ↓yes                                     | Artifact: PRD |
| Parallel|   [Parallel]→[Agent FE][Agent BE]→[Merge]       | Timeout: 24h  |
| Merge   |        ↓                                        | On reject: →  |
| Tool    |   [Test: unit+e2e]→[Approval: Code]→[Deploy]    | Revise node   |
| Test    |                                                 |---------------|
| Deploy  |   minimap · zoom · fit · validation badge       | Run state:    |
|         |                                                 | ✋ waiting 2h  |
|         |                                                 | [Open approval]|
+---------+-------------------------------------------------+---------------+
```

- **Node types (9) `[Đã xác minh]`:** Agent (agent/role, instruction, input/output artifact), Task (tạo task trong Task Center, assignee), Approval (approver, artifact, timeout, on-reject edge), Condition (biểu thức trên output/test result), Parallel (fan-out N nhánh), Merge (join: all / any / N-of), Tool (MCP/tool call với tham số), Test (test suite, pass threshold), Deployment (target, require approval mặc định true).
- **Node visual states `[Đề xuất]`:** `idle` (viền mặc định) · `queued` (viền dashed) · `running` (viền primary + spinner nhỏ + thời lượng) · `waiting_approval` (viền warning + icon ✋) · `blocked` (viền warning + ⛔) · `succeeded` (viền success + ✓ + thời lượng) · `failed` (viền danger + ✗ + nút Retry) · `skipped` (mờ) · `cancelled` (gạch). Edge đang truyền dữ liệu có animation dash chỉ khi run thật.
- **Validation (trước Run/Publish):** không có node mồ côi, Approval có approver, Parallel có Merge tương ứng, Condition có đủ 2 nhánh, Deployment có Approval trước (cảnh báo nếu không) — hiển thị panel lỗi click → focus node.
- **Chế độ:** Design (edit) vs Run view (read-only overlay trạng thái; chọn `?run=`; mở Inspector run: input/output, logs, artifact, duration, cost; nút Retry node / Skip node / Open agent / Open approval).
- **Template:** Discovery → Design → Build → QA → Release (mỗi phase có Approval gate) `[Đề xuất]`.
- **Dữ liệu:** `GET/PUT /api/workflows/:id` (graph JSON), `POST /api/workflows/:id/validate`, `POST /api/workflows/:id/runs`, `GET /api/workflow-runs/:runId` (node states, artifacts), `POST /api/workflow-runs/:id/nodes/:nodeId/{retry|skip}`, WS `workflow.run.node_state`, `workflow.run.*`.
- **Trạng thái:** empty = template picker; run history empty = "Chưa chạy"; version conflict (edit khi đang run) → banner "Run #17 dùng v3; bạn đang sửa v4 draft".
- **Quyền:** Owner edit/run; Viewer xem run.
- **AC:** [ ] 9 node type kéo thả, kết nối theo quy tắc port (Condition 2 out, Merge N in). [ ] 9 visual state phân biệt được cả ở chế độ color-blind (icon + viền, không chỉ màu). [ ] Validate chặn Run khi lỗi. [ ] Trạng thái node cập nhật ≤ 1s sau WS. [ ] Deep link `?run=&node=` mở inspector đúng node. [ ] Undo/redo 50 bước. [ ] Không có animation edge khi không có run.

---

### 3.15 Task Board / Task Center — `/projects/:p/tasks?view=kanban|table|timeline|graph` (Core)

**Mục đích:** Nguồn sự thật về công việc: trạng thái, người làm, phụ thuộc, evidence hoàn thành.

```
+---------------------------------------------------------------------------+
| Tasks · Shop App   [Kanban|Table|Timeline|Graph]  [Search #] [Filters ▾]   |
| Filters: Assignee · Dept · Status · Priority · Phase · Label · Blocked only |
+---------------------------------------------------------------------------+
| KANBAN: Backlog(12) | Ready(5) | In progress(8) | In review(3) | Blocked(2) | Done(31)
| ┌#142 Auth API ───────────┐                                               |
| │ ⛔ Blocked · P1 · Dev-03 │  card: id, title, priority, assignee avatar,  |
| │ 3/7 subtasks · 2 deps   │  evidence chips (subtasks, tests, PR), age    |
| └─────────────────────────┘                                               |
| TABLE: DataTable cột cấu hình, group by, inline edit status/assignee.      |
| TIMELINE: Gantt theo started/estimated(ít tin)/actual; phụ thuộc vẽ mũi tên|
| GRAPH: React Flow dependency DAG; critical path highlight; blocked đỏ.     |
+---------------------------------------------------------------------------+
| Drawer task (?task=): mô tả, acceptance criteria, subtasks, dependencies,  |
|  assignee, evidence (commits, PRs, tests, artifacts), log, comments,       |
|  actions: Assign, Change status, Add dependency, Request review, Split.    |
+---------------------------------------------------------------------------+
```

- **Dữ liệu:** `GET /api/projects/:p/tasks?view=` · `PATCH /api/tasks/:id` · `GET /api/tasks/:id?include=evidence,dependencies,log` · `POST /api/tasks` · WS `task.*`.
- **Hành động:** Create, Assign (gợi ý agent idle + skill match), Drag giữa cột (một số chuyển đổi bị cấm: Backlog → Done cần evidence; cảnh báo), Add dependency, Split, Request review (tạo approval), Bulk (table).
- **Trạng thái:** empty = "Workflow sẽ tạo task tự động" + Create thủ công; timeline empty khi không có started time.
- **Realtime:** card di chuyển cột khi `task.status_changed` (animation ngắn), evidence chip cập nhật.
- **AC:** [ ] 4 view cùng filter, filter trong URL. [ ] Không cho Done thủ công khi task do agent làm chưa có evidence (hiển thị lý do; Owner có thể override với ghi chú `[Đề xuất]`). [ ] Graph hiển thị critical path và vòng phụ thuộc (cảnh báo cycle). [ ] 500 task table scroll mượt (virtualized).

---

### 3.16 Activity Center — `/activity`

```
+---------------------------------------------------------------------------+
| Activity   [All | Needs action | Agents | Tasks | Approvals | Workflows | System]
| [Agent ▾][Project ▾][Severity ▾][Date ▾]   [Mark all read]                 |
| 10:42 ⚠ BA-01 requested approval "PRD v2"            Shop App   [Open]    |
| 10:40 ● Dev-01 committed 3 files to feat/auth (a1b2c3)  Shop App [Changes]|
| 10:38 ⛔ Dev-03 blocked: missing env DATABASE_URL       Shop App [Resolve] |
| … (infinite scroll, group theo ngày)                                       |
+---------------------------------------------------------------------------+
```

- **Dữ liệu:** `GET /api/activity?cursor=&filters` · `POST /api/notifications/read` · WS tất cả category §10 NAVIGATION.
- **AC:** [ ] Mỗi dòng có deep link đúng. [ ] Filter "Needs action" chỉ hiển thị item chưa resolved. [ ] Mark read đồng bộ badge sidebar.

---

### 3.17 Approval Center — `/approvals`, `/approvals/:approvalId?tab=` (Core)

**Mục đích:** Nơi Director ra quyết định với đầy đủ bằng chứng: tài liệu, thiết kế, code diff, test, security.
**Mục tiêu:** Quyết định đúng, nhanh, có ghi vết; yêu cầu sửa với chỉ dẫn rõ.

```
+-------------------------+-------------------------------------------------+
| Inbox (4)  [Pending|All]| PRD v2 — Shop App · from BA-01 · waiting 2h     |
| [Type ▾][Project ▾]     | Workflow: Discovery › Approval: PRD · SLA 24h   |
| ✋ PRD v2       2h  BA-01| Tabs: Summary | Document | Diff (v1→v2) | Tests |
| ✋ Design spec  40m UX-01|       Security | Preview | Conversation | History|
| ✋ PR #12 Auth  12m Dev-01+-------------------------------------------------+
| ✋ Deploy stg   5m  Ops-01| SUMMARY: what changed (agent-written, labeled) |
|                         | Evidence: 12 sections · 3 open questions         |
|                         | Checks: ✓ lint  ✓ unit 118/120  ⚠ security 1 med |
|                         | Risk notes (agent) · Related tasks (#140, #142)  |
|                         +-------------------------------------------------+
|                         | [✓ Approve] [✎ Request changes] [✗ Reject] [💬] |
+-------------------------+-------------------------------------------------+
```

- **Loại approval `[Đề xuất]`:** `document` (BA/PRD/spec), `design` (wireframe/spec/asset preview), `code` (PR/diff), `test_result`, `security_result`, `deployment`, `budget` (vượt ngân sách), `permission` (agent xin quyền ngoài allowlist — tương đương "permission request" ở Pixel Agents/Claude Code hook).
- **Tabs theo loại:** Document (markdown render, outline, comment inline), Diff (before/after: `DiffViewer` side-by-side/inline, file tree, comment theo dòng), Preview (iframe sandbox của Product Preview hoặc ảnh design), Tests (suite, pass/fail, failing test log), Security (findings theo severity, file/line, suggested fix), Conversation (hỏi agent trực tiếp trước khi quyết), History (các version & quyết định trước).
- **Hành động `[Đã xác minh]`:** Approve (confirm nếu loại `deployment`/`permission`), Reject (bắt buộc lý do), Request changes (bắt buộc comment; có inline comments gộp vào; tạo task revise cho agent), Comment (không đổi trạng thái). Secondary: Assign to another agent for second opinion (tạo review task), Snooze (nhắc lại sau X giờ), Open in workspace, Download artifact.
- **Dữ liệu:** `GET /api/approvals?status=&type=&project=` · `GET /api/approvals/:id?include=artifact,diff,checks,history` · `POST /api/approvals/:id/{approve|reject|request_changes|comment}` · WS `approval.*`, `qa.report.ready` (checks update).
- **Trạng thái:** inbox empty = "Inbox zero"; detail loading per-tab; checks pending = "Đang chạy test (42/60)" thật; artifact lớn (diff > 5k dòng) → load theo file, cảnh báo.
- **Quyền:** Owner quyết; Viewer xem + comment `[Đề xuất]`.
- **AC:** [ ] Mọi quyết định ghi history (ai, khi nào, lý do) và gửi event về workflow ≤ 1s. [ ] Reject/Request changes không gửi được khi thiếu lý do. [ ] Diff hỗ trợ ≥ 200 file với file tree và tìm kiếm. [ ] Checks hiển thị trạng thái pending/running/done thật, không hiển thị ✓ giả khi chưa chạy. [ ] Phím `A/X/M` mở form tương ứng, không áp dụng ngay. [ ] Mobile: đủ 3 action + diff theo file.

---

### 3.18 Code Workspace — `/projects/:p/workspace?file=&line=&branch=&panel=` (Core, Monaco) `[Đã xác minh]`

**Mục đích:** Xem và can thiệp vào code agent đang viết: explorer, editor, terminal, git, test, diff, review — không bắt Director mở IDE ngoài.
**Mục tiêu:** Hiểu agent vừa đổi gì, chạy test, sửa nhỏ, review PR, quản lý branch.

```
+---------------------------------------------------------------------------+
| Shop App › Workspace   branch: feat/auth ▾  [Sync][Commit…][Open PR]       |
| Agent activity: ● Dev-01 editing src/api/auth.ts · ● QA-02 running tests  |
+----------+------------------------------------------+---------------------+
| EXPLORER | EDITOR tabs: auth.ts ● | user.ts | README  | RIGHT PANEL (tabs) |
| ▸ src    |  (Monaco; gutter: agent-edited lines có   | Git changes (7)    |
|   ▾ api  |   marker màu agent; lock khi agent đang   |  M src/api/auth.ts |
|     auth.ts●|  ghi file: banner "Dev-01 đang sửa")   |  A src/api/jwt.ts  |
|     user.ts|                                          | Agent activity     |
| ▸ tests  |                                            | Test results 42/60 |
| ▸ docs   |                                            | Logs               |
|          |                                            | Code review (PR#12)|
+----------+------------------------------------------+---------------------+
| BOTTOM PANEL tabs: Terminal (xterm) | Problems | Test output | Diff viewer |
|  Commit history (graph) | Branches                                       |
+---------------------------------------------------------------------------+
```

- **Vùng:** Explorer (tree, search, file đang bị agent sửa có dot màu agent), Editor (Monaco; multi-tab; read-only khi agent giữ lock trên file `[Đề xuất]`; có "Take over" với confirm = pause agent), Right panel (Git changes với stage/unstage/discard, Agent activity per-file, Test results, Logs, Code review: comment theo dòng → gửi thành Request changes cho agent), Bottom panel (Terminal thật gắn sandbox project, Problems, Test output, Diff viewer, Commit history, Branches: create/switch/merge/delete với bảo vệ main).
- **Dữ liệu:** `GET /api/projects/:p/fs/tree`, `GET/PUT /api/projects/:p/fs/file?path=` , `GET /api/projects/:p/git/{status|log|branches|diff}` , `POST /api/projects/:p/git/{commit|checkout|branch|merge}` , `POST /api/projects/:p/terminal` (WS PTY), `GET /api/projects/:p/tests/latest`, WS `fs.changed`, `git.changed`, `agent.tool_event`(file scope), `test.progress`.
- **Hành động:** Open file, Edit/Save (tạo commit có author "Director" tách biệt commit của agent), Run tests, Commit, Create branch, Open PR/approval, Review comment → request changes, Revert file, Take over file.
- **Trạng thái:** repo chưa init → EmptyState "Project chưa có repo — agent sẽ tạo ở bước Build" + Init thủ công; file nhị phân → "Không xem được"; terminal mất kết nối → Reconnect; xung đột ghi → dialog diff 3 bên `[Đề xuất]`.
- **Quyền:** Owner; Viewer read-only (không terminal, không commit).
- **AC:** [ ] Mở file từ deep link đúng dòng. [ ] Agent ghi file đang mở → editor cập nhật và cảnh báo, không mất chỉnh sửa chưa lưu (giữ bản local + diff). [ ] Commit của Director và agent phân biệt được trong history. [ ] Terminal là PTY thật trong sandbox, có giới hạn quyền như agent. [ ] Review comment chuyển thành approval `request_changes` với link dòng. [ ] Monaco load lazy, không chặn render shell.

---

### 3.19 Product Preview — `/projects/:p/preview`

```
+---------------------------------------------------------------------------+
| Preview · Shop App   Build: #23 (feat/auth, a1b2c3) ● running :5173        |
| [Desktop|Tablet|Mobile] [Reload] [Open in new tab] [Console] [Share link]  |
| ┌───────────────────────────────────────────────────────────────────────┐ |
| │ iframe sandbox (dev server / static build do agent chạy trong sandbox)│ |
| └───────────────────────────────────────────────────────────────────────┘ |
| Status: build log tail · errors (console) · [Request changes] [Accept build]|
+---------------------------------------------------------------------------+
```

- **Dữ liệu:** `GET /api/projects/:p/previews` (build id, commit, url, status, log), `POST /api/projects/:p/previews/:id/{start|stop}`, WS `preview.status`, `preview.console`.
- **Hành động:** Switch build/branch, Device frame, Reload, Console, Request changes (tạo approval/revise task kèm screenshot `[Đề xuất]`), Accept build (gate release).
- **Trạng thái:** chưa có build → "Chưa có build; chạy workflow Build hoặc Start preview"; build fail → log lỗi thật + "Ask Dev-01 to fix" (tạo task).
- **AC:** [ ] Preview là app thật chạy trong sandbox, không phải ảnh tĩnh. [ ] Console lỗi runtime hiển thị trong app. [ ] Request changes mang theo URL + screenshot.

---

### 3.20 QA & Security Report — `/projects/:p/qa`, `/projects/:p/qa/runs/:runId?finding=` (Core)

**Mục đích:** Một nơi xem kết quả test, coverage, lint, security scan, với evidence và hành động sửa.

```
+---------------------------------------------------------------------------+
| QA & Security · Shop App   Latest run #42 (a1b2c3) · 10:40 · ⚠ 1 critical  |
| [Run tests][Run security scan][Compare with run ▾]                         |
| ┌ Unit 118/120 ✓ ┐ ┌ E2E 40/42 ⚠ ┐ ┌ Coverage 71% ┐ ┌ Lint 0 ✓ ┐ ┌ Sec 1C 2M 5L ┐
+---------------------------------------------------------------------------+
| Tabs: Tests | Security | Coverage | Trends | History                       |
| SECURITY: Severity ▾ Status ▾ Tool ▾                                       |
| 🔴 Critical · SQL injection · src/api/user.ts:88 · semgrep · open          |
|    Evidence: snippet · rule link · suggested fix                           |
|    [Assign fix → Dev-03][Mark false positive (reason)][Open in workspace]  |
| TESTS: suite tree, failing first, log, flaky marker (fail/pass alternating)|
| TRENDS: pass rate / coverage / findings theo run (chart)                   |
+---------------------------------------------------------------------------+
```

- **Dữ liệu:** `GET /api/projects/:p/qa/runs` · `GET /api/qa/runs/:id?include=tests,findings,coverage` · `POST /api/projects/:p/qa/runs` (`{type: tests|security|all}`) · `POST /api/findings/:id/{assign|dismiss}` · WS `qa.run.progress`, `qa.report.ready`.
- **Hành động:** Run tests/scan (agent QA/Security thực thi), Assign fix (tạo task + link finding), Dismiss với lý do (ghi history), Compare runs, Export report (PDF/MD), Create approval gate.
- **Trạng thái:** chưa có run → "Chưa có kết quả; chạy test hoặc đợi workflow QA"; run đang chạy → progress thật (`42/60`), có Cancel.
- **AC:** [ ] Finding link tới dòng code trong Workspace. [ ] Dismiss cần lý do, có history. [ ] Metric card là số thật từ run; "—" khi không chạy. [ ] Flaky detection dựa trên ≥ 3 run. [ ] Critical finding mở → badge sidebar + Needs-you.

---

### 3.21 Cost Management — `/cost?from=&to=&groupBy=&project=`

```
+---------------------------------------------------------------------------+
| Cost   [Range ▾] [Group by: Agent|Project|Provider|Department|Model ▾]      |
| [Total $184.20][Today $38.40][Tokens 6.1M][Budget 68% used ⚠][Forecast —]  |
| Chart: cost theo ngày (stacked theo groupBy)                               |
| Table: name · tokens in/out · requests · cost · % · trend · [budget ▸]     |
| Budgets & alerts: per project / department / agent · ngưỡng 80% / 100%     |
| Actions on breach: notify only | pause new tasks | pause agents  [Save]    |
+---------------------------------------------------------------------------+
```

- **Dữ liệu:** `GET /api/costs?from&to&groupBy` (từ usage log provider thật, pricing table có nguồn & ngày cập nhật), `GET/PUT /api/budgets`, WS `cost.updated`, `cost.threshold`.
- **AC:** [ ] Mỗi con số có tooltip: nguồn (provider usage) và bảng giá áp dụng (ngày). [ ] Forecast chỉ hiện khi có ≥ 7 ngày dữ liệu, ghi rõ phương pháp. [ ] Hành động khi vượt ngân sách thực thi đúng (pause tasks) và có log.

---

### 3.22 Settings — `/settings/:section`

- **Sections:** General (tên công ty, timezone, default landing Dashboard/Office `[Cần người dùng phê duyệt]`, ngôn ngữ), Appearance (theme light/dark/system, density, reduced motion, sidebar default), Providers (shortcut tới Hub), Security (sandbox mode mặc định, allowlist lệnh shell, secret store, viewer accounts — sau), Notifications (bật/tắt theo category, sound cho `approval.requested`/`agent.blocked`, email/webhook sau), Office (FPS cap, hiển thị bubble, âm thanh, layout mặc định), Shortcuts (xem/đổi), About (version, changelog, diagnostics export).
- **Component:** `SplitPane` (nav trái 220px), `Form` sections, `Switch`, `Select`, `Kbd`.
- **Dữ liệu:** `GET/PATCH /api/settings`, `GET /api/system/info`.
- **AC:** [ ] Mọi setting áp dụng ngay (theme) hoặc nêu rõ "áp dụng từ task tiếp theo". [ ] Export diagnostics không chứa secret.

---

## 4. Multi-agent monitoring — mapping yêu cầu chủ sở hữu `[Đã xác minh]`

| Yêu cầu | Nơi đáp ứng |
|---|---|
| name / role / provider / model / current task / state | AgentCard (§3.6), Agent Panel Office (§3.3), Agent Detail header (§3.7), Dashboard feed (§3.2) |
| evidence-based progress | chip `x/y` với tooltip nguồn (subtasks, tests, nodes); không bao giờ hiển thị % không có mẫu số |
| runtime / last log | AgentCard, Agent Panel, Agent Detail Overview; Logs tab stream |
| pause / resume / stop / retry / reassign | action bar + `⋯` + bulk + command palette + shortcut P/R/S |
| view logs / view changes / open workspace / request review | action bar; deep link tới Logs tab, Diff (Workspace `panel=changes`), Workspace, tạo approval |

## 5. Quyết định & lý do

| Quyết định | Lý do | Nhãn |
|---|---|---|
| "Needs you now" ở đầu Dashboard và Attention banner ở Agents | Attention routing là chiến lược chính chống quá tải 18 agent (xem `UX_RESEARCH_REPORT.md` §5). | `[Đề xuất]` |
| Office không có hoạt động trang trí (pet, TV, coffee, meeting giả) | Vi phạm evidence-driven; gây nhiễu nhận thức "ai thực sự đang làm". Giữ tối thiểu idle animation. | `[Cần người dùng phê duyệt]` (chủ sở hữu có thể muốn "vui" hơn) |
| Progress luôn dạng `x/y` + nguồn | Loại bỏ fake progress theo brief. | `[Đã xác minh]` |
| Editor read-only khi agent giữ lock, "Take over" = pause agent | Tránh xung đột ghi và mất công agent. | `[Đề xuất]` |
| Approval Request changes bắt buộc comment & tạo task revise | Vòng lặp sửa phải có đầu vào rõ cho agent. | `[Đề xuất]` |
| Không cho Done thủ công task của agent khi thiếu evidence (Owner override có ghi chú) | Giữ dữ liệu progress đáng tin. | `[Đề xuất]` |
| Office lỗi → fallback list | Office là lớp phụ, không được chặn công việc. | `[Đề xuất]` |

## 6. Dependencies
- Agent 11: component list ở đầu tài liệu; token status; chart style.
- Agent Backend: endpoint/entity `[Giả định]` trong từng mục; đặc biệt `attention`, `evidence`, `provider.health`, `workflow-runs.node_state`.
- Agent Realtime: WS topic; throttle guidelines (500ms/agent feed, 1s node state).
- Agent Sandbox/Security: PTY terminal, file lock, secret masking, permission mode.
- Agent Office/Game: engine chọn (PixiJS/Canvas), sprite set tự vẽ (license), layout JSON schema riêng.

## 7. Rủi ro
- Khối lượng 22 màn + 10 tab Agent Detail lớn hơn Phase 3 cho phép → ưu tiên 11 core; các màn còn lại có thể là "embed/lite" của màn core (Project Detail nhúng Tasks/Approvals).
- Office engine và Monaco/React Flow cùng lúc → bundle lớn; cần code-split theo route.
- Realtime 18 agent × tool events có thể hàng chục event/giây → mọi list phải virtualized + throttle.
- Người dùng có thể kỳ vọng Office "vui" như MVO; quyết định tối giản cần được phê duyệt sớm.

## 8. Vấn đề chưa giải quyết
1. Permission mode `full` (không hỏi) có được phép chọn trong UI không?
2. Office: cho phép hoạt động trang trí nhẹ ở trạng thái idle (đi lấy nước) nếu không ảnh hưởng nhận thức?
3. Giới hạn 18 agent: chặn tạo thứ 19 hay cho tạo offline?
4. Reveal secret trong Provider Settings: cho phép hay chỉ rotate?
5. Viewer được comment trong Approval không?

## 9. Acceptance Criteria tổng (áp dụng mọi màn)
- [ ] Có đủ 5 trạng thái: loading (skeleton), empty (CTA), error (retry + mã), success, offline (stale timestamp).
- [ ] Không hiển thị số liệu/progress không có nguồn; "—" thay cho 0 giả.
- [ ] Mọi action mutation có feedback (toast) từ response thật; action phá hủy có confirm.
- [ ] Deep link mở đúng trạng thái; back/forward không hỏng.
- [ ] Realtime cập nhật ≤ 2s, không layout shift, virtualized khi > 50 dòng.
- [ ] Đạt checklist `USABILITY_AUDIT_CHECKLIST.md` trước khi QA ký.


> **Phụ lục sửa đổi:** xem `docs/design/HERDR_DESIGN_INTEGRATION.md` (attention state, Agent Rail, token bổ sung). Khi mâu thuẫn, phụ lục thắng cho các hạng mục đó cho đến khi chủ sở hữu duyệt (D-017..D-019).
