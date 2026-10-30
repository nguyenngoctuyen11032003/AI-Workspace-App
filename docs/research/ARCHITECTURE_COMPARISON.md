# ARCHITECTURE_COMPARISON — So sánh kiến trúc 4 repo tham chiếu

> Tác giả: Technical Research Lead (tổng hợp Phase 0). Ngày: 2026-10-30.
> Nguồn: `docs/research/{CHATDEV,PIXEL_AGENTS,MY_VIRTUAL_OFFICE,OPENHANDS}_ANALYSIS.md`, `LICENSE_AND_REUSE.md`, `CLAUDE_CODE_INTEGRATION.md`, `CODEX_INTEGRATION.md`, `ORCHESTRATION_WORKFLOW_RESEARCH.md`, `MULTI_AGENT_SCALABILITY.md`, `DECISION_LOG.md` (ở thư mục gốc repo, không nằm trong `docs/`), `docs/DATABASE_DESIGN.md`.
> Nhãn: **[Đã xác minh]** = có bằng chứng từ việc đọc mã/chạy lệnh thật trong Phase 0 (kể cả đối chiếu của tài liệu này với repo tham chiếu); **[Đề xuất]** = suy luận/khuyến nghị, chưa có bằng chứng chạy thật.

---

## 1. Mục đích

Đặt 4 repo tham chiếu cạnh nhau theo cùng một bộ tiêu chí để (a) biết mỗi repo thực sự làm gì và không làm gì, (b) xác định phần nào AI WORKSPACE nên tái sử dụng, tham khảo hay viết mới, (c) làm đầu vào cho `REUSE_DECISIONS.md` và `TECHNOLOGY_EVALUATION.md`.

## 2. Phạm vi

- Trong phạm vi: 4 repo tại `D:\Code\Tuyen\Agent management\{ChatDev, pixel-agents, my-virtual-office, OpenHands}`; 13 tiêu chí so sánh; phần ChatDev 1.0 vs 2.0; hàm ý cho AI WORKSPACE.
- Ngoài phạm vi: thiết kế chi tiết (nằm ở `docs/architecture/*`), mã ứng dụng (Phase 0 chỉ có tài liệu), tư vấn pháp lý.
- Giả định sản phẩm: AI WORKSPACE là web app + backend NestJS, closed-source (giả định của `LICENSE_AND_REUSE.md`, chưa được chủ sở hữu xác nhận).

## 3. Bốn repo trong một đoạn

| Repo | Thực chất là gì (đã đối chiếu mã) |
|---|---|
| ChatDev (commit `a2ea4a2`) | `main` là **ChatDev 2.0 "DevAll"**: runtime điều phối workflow bằng YAML (Python 3.12 + FastAPI + Vue 3/Vue Flow). Bản 1.0 "Virtual Software Company" còn ở branch `origin/chatdev1.0` và tag `v1.0.0..v1.1.6`. Ngoài ra repo có branch `macnet`, `puppeteer` và tag `v2.0.0..v2.2.0`. **[Đã xác minh]** |
| pixel-agents v1.5.0 | VS Code extension + CLI `npx pixel-agents`; quan sát Claude Code bằng hook + tail JSONL; vẽ văn phòng pixel bằng Canvas 2D thuần (không Phaser). **[Đã xác minh]** |
| my-virtual-office v0.7.4 | Ứng dụng Python một tiến trình khổng lồ (`app/server.py` ~22k dòng, `app/game.js` ~17k dòng), AGPL-3.0, điều khiển nhiều provider CLI, văn phòng vẽ procedural, không có DB. |
| OpenHands (thực tế là `@openhands/agent-canvas` 1.34.0) | **Không còn là framework Python OpenHands cũ.** Toàn bộ Python đã bị xoá khỏi repo; còn lại là UI React/TypeScript + launcher Node + Electron shell, điều khiển backend Python `openhands-agent-server` (repo `OpenHands/software-agent-sdk`, cài bằng `uvx`, không vendored). **[Đã xác minh]** (package.json ghi MIT, `packageManager: npm@11.12.1`). |

Hệ quả quan trọng: "OpenHands runtime/Docker sandbox/event stream" mà đề bài nhắc tới **không nằm trong repo đã clone**; các nhận định về chúng dựa trên mirror TypeScript của wire contract và tài liệu của repo, không phải trên mã Python gốc. Xem Rủi ro R-03.

---

## 4. Ma trận so sánh

### 4.1 Mục đích, stack, mô hình agent

| Tiêu chí | ChatDev 2.0 | Pixel Agents | My Virtual Office | OpenHands (Agent Canvas) |
|---|---|---|---|---|
| **Purpose** | Engine chạy workflow đa agent mô tả bằng YAML (DAG + cycle + dynamic map/tree); 1.0 là chuỗi phase "công ty phần mềm ảo" | Quan sát session Claude Code và hiển thị mỗi session thành một nhân vật pixel trong văn phòng | "Văn phòng ảo" quản lý nhiều agent provider, kanban, meeting, workflow Backlog→Review→Done | UI + launcher cho Agent Server: tạo/điều khiển conversation, đa backend, desktop shell |
| **Stack** | Python 3.12, FastAPI, WebSocket/SSE, Vue 3 + Vue Flow; Docker/compose | TypeScript; `core/` + `server/` (Fastify) + `adapters/vscode` + `webview-ui` (React 19, Canvas 2D); build esbuild/Vite | Python (http server tự viết), JS thuần + canvas, Docker compose, Kasm browser tùy chọn | React/TypeScript, Zustand, TanStack Query, Monaco, Vite; Node launcher; Electron; Helm chart; backend thực là Python (ngoài repo) |
| **Agent model** | Agent = **node** `agent` trong đồ thị; vai trò nhúng trong node; 1.0: roster `recruitments` 9 vai + `RolePlaying` 2 agent đối thoại | Agent = **một session Claude Code** (1:1 với một nhân vật); sub-agent/teammate phân biệt theo "có tên hay không" | Agent = **profile của provider** (không có bảng Employee riêng); persona được materialize thành file trong workspace | Hai `agent_kind` (LLM agent và ACP agent); **Agent Profiles** tái sử dụng; delegation = child conversation |
| **Giá trị đáng học** | Schema YAML, vòng lặp có điều kiện, edge trigger/condition | Chuẩn hoá hành vi CLI thành `AgentEvent` | Ý tưởng capability flags, presence, workflow review (chỉ ý tưởng) | Event envelope + payload, session frame, client tools |

### 4.2 Giao tiếp, session/state, execution

| Tiêu chí | ChatDev 2.0 | Pixel Agents | My Virtual Office | OpenHands |
|---|---|---|---|---|
| **Communication** | Message queue trên **edge** (`trigger`, `keep_message`, `carry_data`, `clear_context`); 1.0: hội thoại 2 vai tới khi gặp `<INFO>`; human qua WS prompt | Không có giao tiếp giữa agent; chỉ có luồng event CLI → server → UI (hub-and-spoke) | A2A request-reply **đồng bộ**, hiển thị dạng bubble; meeting 1on1/group, bắt buộc có summary | Parent/child conversation; child nhận brief tự chứa, không thấy lịch sử parent |
| **Session / state** | `WorkflowSessionStore` **in-memory**; `WareHouse/` trên đĩa; **không có checkpoint/resume** **[Đã xác minh]** | Per-process `AgentStateStore` toàn cục; layout/config là file JSON; không multi-tenant | **Không có DB**: toàn bộ là file trong `VO_STATUS_DIR`; session theo provider | Conversation do Agent Server giữ, persist file JSON; resume bằng `after_seq`; UI có Zustand store |
| **Execution / sandbox** | Opt-in qua biến môi trường; node `python`/`execute_code`/`uv_run` chạy subprocess, `shell=True` ở 1.0, **không sandbox** | Không thực thi gì; chỉ spawn terminal `claude --session-id` (nếu dùng launcher) và đọc kết quả | **Không có sandbox do MVO sở hữu**; agent chạy với quyền host; Kasm browser là container riêng, tùy chọn | Ba mức sandbox: local, Docker per-conversation (giới hạn CPU/mem/pids), all-in-one image; tmux bash, file editor, browser, VSCode |
| **Provider integration** | `ModelProvider` registry: OpenAI (chat/responses), Gemini, `base_url` tùy ý; **không** có CLI agent | `HookProvider` (hook + transcript) — chỉ Claude hiện thực; Codex/Gemini là roadmap | Provider SDK v1.1 + manifest 22 capability flag: Claude Code (`claude -p stream-json`), Codex (`app-server` JSON-RPC), Hermes, OpenClaw, OpenCode, Antigravity | LLM qua LiteLLM-kiểu cấu hình ở Agent Server; ACP cho phép chạy Claude Code/Codex làm "host" trong sandbox |

### 4.3 Event, rendering, skills, persistence, testing, license

| Tiêu chí | ChatDev 2.0 | Pixel Agents | My Virtual Office | OpenHands |
|---|---|---|---|---|
| **Event system** | `EventType` enum (NODE_START/END, MODEL_CALL, TOOL_CALL, HUMAN_INTERACTION, …) + WS message types (`log`, `human_input_required`, `artifact_created`, `workflow_completed`…) | `AgentEvent` 10 kind (`toolStart`, `toolEnd`, `turnEnd`, `subagent*`, `permissionRequest`, `sessionStart/End`…); AsyncAPI 3.0 cho wire protocol | Presence coordinator với vocabulary lifecycle (`run_started`, `tool_started`, `approval_requested`…) → 6 visual state; client **poll** 3–5 s | Event envelope (`id` ULID, `timestamp`, `source`) + payload có `kind`; Action/Observation ghép bằng `tool_call_id`/`action_id`; durable vs transient vs delta |
| **Office rendering** | Không có; 12 sprite nhân vật gán theo `node_id` trong UI Vue; visualizer 1.0 là Flask replay log | **Canvas 2D** tile 16px, BFS 4 hướng, FSM `idle/walk/type`, z-sort theo Y, sprite cache, layout editor đầy đủ (undo 50 bước) | `game.js` canvas procedural (nhân vật vẽ bằng code), A* lưới, furniture catalog | Không có office; chỉ canvas workflow/chat |
| **Skills / plugins** | `SKILL.md` (frontmatter `name/description/allowed-tools`), 3 loại tooling (function, MCP remote/local, skill), registry pattern | Không có skill; có consent gate khi ghi hook vào `settings.json` của Claude | Skills library, SKILL.md built-in (nội dung AGPL), office-seeded MCP tools | Skills/microagents, `@openhands/extensions`, MCP, plugins, hooks (`hooks.json` giống Claude Code) |
| **Persistence** | WareHouse + JSON log/token; session RAM | File JSON (atomic write) trong `~/.pixel-agents` | File trên đĩa, Markdown cho project/task | File JSON của Agent Server; UI giữ trạng thái client |
| **Testing** | `tests/` pytest, đáng chú ý 3 file test bảo mật (path traversal, attachment, code execution) | Vitest (server + webview, `vitest ^4.1.10`), Playwright (`@playwright/test ^1.58.2`), mock Claude runner `e2e/fixtures/mock-claude-runner.cjs` | `tests/*.py` + `*.js/*.mjs` (review parser, session durability, approval, concurrency) | Vitest 5.0.0 + coverage-v8, Playwright 1.63.0 (nhiều config mock-llm), Stryker mutation, Testing Library **[Đã xác minh từ package.json]** |
| **License** | Apache-2.0 (`Copyright 2025 OpenBMB`, không có NOTICE) | MIT (`Copyright (c) 2026 Pablo De Lucca`); art nhân vật dựa trên JIK-A-4 MetroCity | **AGPL-3.0-or-later** (+ bán license thương mại riêng) | MIT (`Copyright © 2025 OpenHands contributors`); Agent Server (repo khác) **chưa kiểm tra** |

Ghi chú xác minh: license của 3 repo đầu và OpenHands đã được đối chiếu trực tiếp với file `LICENSE`/`package.json` trong lần rà soát này **[Đã xác minh]**.

---

## 5. ChatDev 1.0 vs 2.0

### 5.1 Hai hệ thống khác nhau, cùng một thương hiệu

| Khía cạnh | ChatDev 1.0 (Virtual Software Company) | ChatDev 2.0 (DevAll) |
|---|---|---|
| Mô hình thực thi | Chuỗi phase tuyến tính `ChatChain`; `ComposedPhase` lặp `cycleNum` cố định | Đồ thị YAML tổng quát (DAG + cycle + subgraph + dynamic map/tree) |
| Đơn vị công việc | Phase = một cuộc đối thoại 2 vai (instructor ↔ assistant) qua `RolePlaying` | Node = một lần gọi LLM hoặc human/python/literal/passthrough/loop_counter |
| Cấu hình | 3 file JSON: `ChatChainConfig.json`, `PhaseConfig.json`, `RoleConfig.json` + class Python cho từng phase | 1 file YAML tự mô tả; không cần code để thêm workflow |
| Khái niệm "công ty" | Có: roster `recruitments`, 9 vai (CEO, CPO, CTO, CHRO, Counselor, Programmer, Reviewer, Tester, CCO) | Không có; vai trò chỉ là prompt trong node |
| State chung | `ChatEnv.env_dict` mutable dùng chung (`task_prompt`, `ideas`, `codes`, `test_reports`…) | Message trên edge + memory store |
| Điều kiện dừng | `<INFO>` + `break_cycle()` viết bằng code | `keyword`/`function` condition, `loop_counter`, `loop_timer` |
| Tool | Không; LLM xuất code block rồi parse | Function calling + MCP + Agent Skills |
| Provider | Chỉ OpenAI chat, enum cứng | Registry; OpenAI/Gemini; `base_url` tùy ý |
| Human-in-loop | Phase `CodeReviewHuman` dùng `input()` | Node `human` + WS prompt + attachments, timeout 1800 s |
| API | Không | REST + WS + SSE + Python SDK |
| Frontend | Flask log viewer + replay | Vue 3 + Vue Flow builder, launch console, batch run |
| Cách tìm thấy | `git show v1.1.6:CompanyConfig/Default/*.json` hoặc branch `chatdev1.0` | Working tree `main` |

2.0 tự "port" 1.0 thành `yaml_instance/ChatDev_v1.yaml` (1021 dòng) với `loop_counter` và edge `keyword: <INFO>`; điều này cho thấy mô hình phase có thể biểu diễn bằng graph mà không mất ngữ nghĩa chính.

### 5.2 Hệ quả cho AI WORKSPACE

1. **Nội dung có giá trị nhất nằm ở 1.0**: role prompt và phase prompt. Chúng không có trong working tree `main`; phải lấy từ tag `v1.1.6`/branch `chatdev1.0`. Ghi rõ nguồn commit/tag khi trích để đáp ứng yêu cầu attribution Apache-2.0.
2. **Cấu trúc có giá trị nhất nằm ở 2.0**: schema YAML (`nodes/edges/start/end/vars`, edge `trigger/condition/carry_data/keep_message/clear_context/process/dynamic`), vòng lặp bằng Tarjan SCC + super-node, `EventType`, protocol WS.
3. **Đề xuất sản phẩm [Đề xuất]**: phơi bày "Company pipeline template" (UX đơn giản kiểu 1.0) nhưng sinh ra graph kiểu 2.0 bên dưới; không cố lặp lại `RolePlaying` hai agent vì agent của ta (Claude Code/Codex) đã có tool thật.
4. **Điều ChatDev không có và ta phải làm**: durable checkpoint/resume, sandbox thật, persistence PostgreSQL, multi-tenant. `ORCHESTRATION_WORKFLOW_RESEARCH.md` §0 nêu rõ đây là điểm phân biệt "Conductor" với ChatDev.
5. **Lưu ý license**: dùng nội dung prompt/YAML của ChatDev hợp lệ theo Apache-2.0 nếu giữ attribution và ghi chú thay đổi; không dùng tên/logo ChatDev. Bộ 12 sprite trong `frontend/public/sprites/` **chưa có thông tin nguồn gốc/license riêng** — xem `REUSE_DECISIONS.md`.

---

## 6. Hàm ý cho AI WORKSPACE

### 6.1 Điều chắc chắn rút ra

1. **Không repo nào cung cấp sẵn thứ ta cần.** ChatDev có engine nhưng là Python, in-memory, không sandbox. Pixel Agents có quan sát Claude + office nhưng 1 session = 1 nhân vật, không có domain công ty/task. MVO có đủ tính năng nhưng AGPL. OpenHands có runtime/sandbox mạnh nhưng nằm ngoài repo và là Python.
2. **Boundary "Provider" là hạt nhân.** Pixel Agents (`HookProvider`/`AgentEvent`), MVO (capability flags) và OpenHands (ACP, `agent_kind`) đều hội tụ về một interface adapter + một event union chuẩn. Đây là nơi AI WORKSPACE nên đầu tư thiết kế: `AgentBrainAdapter` với `ClaudeCodeAdapter`, `CodexAdapter`, `OpenHandsProvider`.
3. **Event model nên mượn từ OpenHands, không từ Pixel Agents**: tách envelope (id/seq/timestamp/source) khỏi payload có `kind`; ghép action↔observation bằng id; tách durable/transient/delta; resume bằng `after_seq`/`lastEventId`. Pixel Agents' `toolId:'current'` là workaround phải tránh (`PIXEL_AGENTS_ANALYSIS.md` §3.3).
4. **Quan sát Claude Code theo mô hình "hook push + transcript pull"** của Pixel Agents là bổ sung chứ không thay thế SDK: với `@anthropic-ai/claude-agent-sdk` (D-004) ta nhận message typed trực tiếp; hook/transcript chỉ cần cho trường hợp session do người dùng tự chạy ngoài app.
5. **Office rendering**: engine Pixel Agents (BFS, FSM, layout model, manifest asset) tách khỏi renderer; D-011 chọn Phaser nên giữ logic, thay renderer. Chuyển động chỉ được điều khiển bởi event thật.
6. **Workflow engine tự xây ("Conductor", D-003)** học từ ChatDev về layered-topological execution và edge semantics nhưng khác ở durable state trong PostgreSQL và approval không giữ worker.

### 6.2 Bảng quyết định theo năng lực

| Năng lực của AI WORKSPACE | Học từ | Tái sử dụng/viết mới | Ghi chú |
|---|---|---|---|
| Workflow definition + validate | ChatDev 2.0 | Viết mới (Zod/TS) giữ tên field tương thích | Import được `yaml_instance/*.yaml` |
| Role/phase mặc định | ChatDev 1.0 | Tái sử dụng nội dung prompt, có chỉnh | Bỏ ràng buộc code-block |
| Claude Code adapter | Pixel Agents (hook/transcript) + tích hợp thật của A05 | Viết mới trên SDK; copy có chọn lọc phần hook/tail | Đã smoke-test SDK dùng OAuth login **[Đã xác minh bởi A05]** |
| Codex adapter | MVO (ý tưởng `app-server`), A06 | Viết mới | **Chưa chạy thật**: Codex CLI chưa cài trên máy |
| OpenHands provider | OpenHands | Dùng `@openhands/typescript-client` + Agent Server như service ngoài | Chưa thử chạy; cần verify license Agent Server |
| Event store/stream | OpenHands, ChatDev | Viết mới (Postgres append-only + outbox + Socket.IO) | D-014 |
| Sandbox | OpenHands, ChatDev (anti-pattern) | Viết mới (Docker) hoặc tận dụng Agent Server | D-007 hai tầng |
| Virtual Office | Pixel Agents | Tái sử dụng logic engine MIT, thay renderer Phaser, thay asset | Art nhân vật cần xác minh/thay |
| Team/task/approval domain | MVO (ý tưởng) | Viết mới hoàn toàn, clean-room | Không đọc mã MVO khi viết |
| Desktop shell | OpenHands (Electron) | Tham khảo, triển khai bằng Tauri ở phase sau | D-001 |

### 6.3 Mâu thuẫn giữa các tài liệu nghiên cứu (cần chủ sở hữu biết)

| # | Mâu thuẫn | Chi tiết | Hướng xử lý đề xuất |
|---|---|---|---|
| C-1 | ORM | `ORCHESTRATION_WORKFLOW_RESEARCH.md` §3.2 viết schema bằng **Prisma** và code mẫu `prisma.$transaction`; `DATABASE_DESIGN.md` §1 và D-002 chọn **Drizzle**; `AGENT_ORGANIZATION.md` còn ghi đường dẫn `prisma/**` | Coi Drizzle là chuẩn (D-002); schema trong tài liệu orchestration chỉ là minh hoạ, cần dịch sang Drizzle khi triển khai |
| C-2 | Giấy phép art JIK-A-4 | `PIXEL_AGENTS_ANALYSIS.md` §5.3 nói **chưa rõ**, cần kiểm tra itch.io; `LICENSE_AND_REUSE.md` §6.1 nói **CC0 1.0 đã xác minh** từ trang itch.io (qua web search, không thấy bằng chứng lưu lại) | Xếp vào "Đề xuất": coi là CC0 nhưng chụp trang license + lưu bằng chứng trước khi bundle |
| C-3 | Concurrency mặc định | `CLAUDE_CODE_INTEGRATION.md` §6 đề xuất `MAX_PARALLEL_AGENTS` mặc định **10**; `ORCHESTRATION` semaphore global **10**; `CODEX_INTEGRATION.md` §9 đề xuất **2–3** turn/account; D-010 chọn **5** mặc định, tối đa 18 | D-010 là quyết định mới nhất; per-provider cap phải nhỏ hơn global (Codex 2–3) |
| C-4 | Renderer office | `PIXEL_AGENTS_ANALYSIS.md` §8.1 khuyến nghị dùng thẳng `renderer.ts` (Canvas); `LICENSE_AND_REUSE.md` §5.1 và D-011 chọn **Phaser** | Giữ logic engine, bỏ `renderer.ts`/`spriteCache.ts` nếu chọn Phaser |
| C-5 | Test framework khi port test bảo mật ChatDev | `CHATDEV_ANALYSIS.md` §11.1 nói port sang **Jest**; stack frontend/đề bài dùng **Vitest** | Chọn Vitest cho frontend/shared; backend NestJS cần quyết định (xem TECHNOLOGY_EVALUATION) |
| C-6 | Phạm vi "tái sử dụng" OpenHands | `LICENSE_AND_REUSE.md` §3.1 ghi "ALLOW … UI component, hooks"; `OPENHANDS_ANALYSIS.md` §11.3 nói UI gắn chặt HeroUI/i18n/PostHog/Cloud → **không** nên copy | Coi UI là reference-only; chỉ tái sử dụng contract/client/ingress/process utils |
| C-7 | Vị trí DECISION_LOG | Đề bài tham chiếu `docs/DECISION_LOG.md`; file thực tế ở `AI-Workspace-App/DECISION_LOG.md` | Giữ nguyên vị trí, sửa tham chiếu |
| C-8 | Hình thức tích hợp Claude Code | `CHATDEV_ANALYSIS.md` §11.3 gợi ý spawn `claude -p --output-format stream-json`; D-004 chọn Agent SDK, raw CLI chỉ là fallback | D-004 thắng; ghi chú trong REUSE_DECISIONS |

---

### 6.4 Chi tiết theo tiêu chí (ghi chú đọc mã)

**Agent model.**
- ChatDev 2.0: agent node có stage pre/gen/post trong `runtime/node/executor/agent_executor.py`, hỗ trợ memory hook và thinking/reflection; không có định danh agent bền vững giữa các run.
- Pixel Agents: định danh agent = `sessionId` của Claude Code; sub-agent không tên được theo dõi trong shadow `AgentStateStore` (`server/src/subagentWatch.ts`) rồi dịch thành `subagentToolStart/Done`.
- MVO: persona là file materialize vào workspace; không có Employee table; "PM" là một vòng lặp Python chứ không phải agent thật.
- OpenHands: `agent_kind` ∈ {LLM, ACP}; Agent Profile là bản định nghĩa có thể tái sử dụng; system prompt và skills/microagents nằm ở Agent Server.
- Bài học cho ta: tách Agent Identity / Provider Identity / User Identity (D-016); 18 vai trò mặc định là seed data.

**Communication.**
- ChatDev: thông điệp đi qua cạnh, có cờ `trigger`, `keep_message`, `clear_context`; với dynamic edge có `split` theo message/regex/json_path.
- OpenHands: child conversation chỉ nhận brief tự chứa; đây là mẫu cho "token economy" của message bus (`ORCHESTRATION_WORKFLOW_RESEARCH.md` §5).
- MVO: A2A đồng bộ, không có bus; AI WORKSPACE cần thread/bus bất đồng bộ.
- Pixel Agents: không có (mỗi session độc lập).

**Session / state.**
- Điểm chung: cả 4 repo lưu state bằng file hoặc RAM; **không repo nào** dùng database quan hệ. Đây là khoảng trống lớn nhất mà PostgreSQL (D-002, 44 bảng) lấp.
- OpenHands có mô hình resume tốt nhất: `after_seq` + frame durable; ChatDev có session snapshot nhưng mất khi tiến trình chết.

**Execution / sandbox.**
- ChatDev 1.0 chạy `python main.py` 3 giây bằng `shell=True` (`exist_bugs()`); 2.0 cho opt-in toàn cục qua biến môi trường nhưng vẫn không sandbox.
- OpenHands là repo duy nhất có sandbox thật (Docker per-conversation, giới hạn CPU/mem/pids) nhưng thuộc Agent Server, không nằm trong repo đã clone.
- Hệ quả cho D-007: Tier 1 (CLI chạy host + policy + Docker cho build/test) phải tự viết; Tier 2 có thể mượn Agent Server.

**Provider integration.**
- Chỉ MVO thực sự tích hợp CLI Claude Code và Codex (bằng subprocess `stream-json` và `app-server` JSON-RPC) nhưng bị cấm copy; Pixel Agents chỉ quan sát Claude; OpenHands đưa ra khái niệm ACP để chạy các CLI này bên trong sandbox.
- Tích hợp CLI của ta dựa trên tài liệu chính thức và smoke test của A05/A06 chứ không dựa trên mã MVO.

**Event system.**
- ChatDev có `EventType` mức workflow; Pixel Agents có `AgentEvent` mức tool; OpenHands có Action/Observation mức LLM; MVO có presence mức hiển thị. Event của AI WORKSPACE cần ba tầng: (1) event nguồn từ provider (chuẩn hoá thành union), (2) event nghiệp vụ (task/approval/workflow), (3) event hiển thị cho Office (suy ra từ 1 và 2).
- Quy tắc từ OpenHands: giữ `seq` tăng dần; tách durable/transient/delta; bỏ qua kind lạ.

**Office rendering.**
- Chỉ Pixel Agents có engine tái sử dụng được và có layout editor; MVO có engine procedural AGPL.
- Mô hình dữ liệu cần giữ: `OfficeLayout` JSON (tile, furniture, seat, area), manifest asset (rotation→state→animation).
- Trạng thái nhân vật Pixel Agents chỉ có `idle/walk/type` (+ frame `read`); ta cần thêm `thinking`, `talking`, `error`, `celebrate`, `waiting_approval`.

**Skills / plugins.**
- Chuẩn SKILL.md/Agent Skills là điểm hội tụ của ChatDev, OpenHands, MVO và Claude Code; D-013 dùng chuẩn này, mở rộng bằng `skill.yaml` và `metadata.aiw-*`.
- Nội dung skill tích hợp sẵn của MVO không được copy; skill mẫu của ChatDev/OpenHands cần kiểm tra license từng skill.

**Persistence.**
- Pixel Agents: ghi file nguyên tử (atomic write) cho layout/config — nên giữ cho việc export layout.
- OpenHands: secrets mã hoá qua `X-Expose-Secrets: encrypted`; D-008 của ta chọn OS keychain và không lưu secret trong Postgres.

**Testing.**
- Mẫu đáng học nhất: mock Claude runner của Pixel Agents (không tốn quota), test bảo mật path/attachment của ChatDev, e2e `mock-llm` và `bind-policy` của OpenHands.
- Phạm vi test của MVO (review parser, session durability, approval, concurrency) là danh sách ý tưởng ca kiểm thử.

**License.**
- Apache-2.0 (ChatDev): giữ license, ghi thay đổi, giữ NOTICE nếu có.
- MIT (Pixel Agents, OpenHands): giữ copyright + văn bản license.
- AGPL-3.0 (MVO): copyleft lây qua mạng (§13); chỉ ý tưởng.
- Asset: license riêng, xem `REUSE_DECISIONS.md` §4 và §6.

### 6.5 Hai luồng dữ liệu tham chiếu

Luồng quan sát (Pixel Agents), phù hợp khi session Claude do người dùng tự khởi chạy:

```
Claude Code ──hook(12 event)──▶ claude-hook.js ──HTTP POST 127.0.0.1──▶ server ──AgentEvent──▶ WS ──▶ webview
     └──────── JSONL transcript (poll 500 ms) ───────────────────────────▶ server (nội dung tool, context usage)
```

Luồng điều khiển (đề xuất cho AI WORKSPACE, kết hợp Agent SDK và event store):

```
React UI ──REST/Socket.IO──▶ NestJS (Conductor scheduler)
                               ├─▶ AgentBrainAdapter.run() ──▶ Claude Agent SDK / Codex app-server / OpenHands
                               │        └─ stream → chuẩn hoá AgentEvent → Postgres (agent_events) + outbox
                               └─▶ outbox ──▶ Redis pub/sub / Stream ──▶ Socket.IO ──▶ UI + Phaser Office
```

Điểm khác biệt chính: Pixel Agents **quan sát** tiến trình do người khác điều khiển; AI WORKSPACE **sở hữu** vòng đời (spawn, approval, cancel, resume), vì vậy cần `canUseTool`/hook chặn thay vì chỉ đọc log.

### 6.6 Những gì cả bốn repo đều không có

1. Persistence quan hệ và event store phân vùng.
2. Multi-tenant, RBAC, audit.
3. Task board có phụ thuộc + PM agent thật + reviewer độc lập chạy test.
4. Cô lập git worktree cho mỗi agent (D-009) và manifest quyền sở hữu file.
5. Quản lý quota subscription (đọc `rate_limit_event`, `account/rateLimits/read`) và host guard.
6. Chuyển động văn phòng điều khiển hoàn toàn bằng event thật (D-011).
7. Resume workflow từ node lỗi (ChatDev không có checkpoint).

---

## 7. Rủi ro

| ID | Rủi ro | Mức | Giảm thiểu |
|---|---|---|---|
| R-01 | Nhầm lẫn "OpenHands" với framework Python cũ dẫn tới ước lượng sai công sức tích hợp | Cao | Ghi rõ trong tài liệu; spike tích hợp Agent Server trước khi cam kết D-006 |
| R-02 | Wire contract của Agent Server và format JSONL/hook của Claude Code thay đổi nhanh theo version | Cao | Version-gate, fixture test, ghim version (`1.59.0`, Claude Code 2.1.312, SDK 0.3.314 là mốc đã ghi) |
| R-03 | Agent Server là code Python ngoài repo, license chưa kiểm tra | Trung bình | Đọc license repo `software-agent-sdk` trước khi triển khai D-006 |
| R-04 | Lây nhiễm AGPL nếu ai đó tham khảo MVO rồi viết lại gần như 1:1 | Cao | Quy trình clean-room (người viết spec ≠ người viết code), không mở file MVO khi code |
| R-05 | Art nhân vật và font của Pixel Agents có nguồn gốc chưa rõ | Trung bình | Chỉ bundle asset có `CREDITS.json` + license rõ; thay font FS Pixel Sans |
| R-06 | Các kết luận về Codex chưa được chạy thật | Trung bình | Chạy live test sau khi cài CLI (`CODEX_INTEGRATION.md` §11.4) |
| R-07 | Kết luận so sánh dựa trên tài liệu phân tích do nhiều agent viết, chỉ đối chiếu chọn lọc | Thấp | Đã đối chiếu license, test framework, cấu trúc thư mục; phần còn lại giữ nhãn nguồn |

## 8. Vấn đề chưa giải quyết

1. License của `openhands-agent-server`/`software-agent-sdk` và `@openhands/typescript-client` (npm) — chưa kiểm tra.
2. Nguồn gốc và license của 12 sprite ChatDev và furniture/floor/wall của Pixel Agents (README chỉ credit JIK-A-4 cho nhân vật).
3. Mức độ ổn định của `codex app-server` (đánh dấu `[experimental]`) — chưa chạy thật.
4. Quota/ToS của subscription khi chạy nhiều session song song (DECISION_LOG mục "Mâu thuẫn và khoảng trống" #2).
5. ChatDev: tag `v2.x` tồn tại cùng `main`; chưa xác định commit nào của 2.x ổn định nhất để làm mốc tham chiếu schema.

## 9. Cần người dùng phê duyệt

- [ ] Xác nhận giả định "AI WORKSPACE là sản phẩm closed-source" (ảnh hưởng mọi verdict license).
- [ ] Xác nhận chính sách clean-room cho My Virtual Office (reference-only).
- [ ] Xác nhận hướng "Company pipeline template sinh graph" thay vì giữ hai mô hình thực thi (phase và graph).
- [ ] Chọn xử lý mâu thuẫn C-1…C-8 theo hướng đề xuất ở §6.3 (hoặc chỉ định hướng khác).
- [ ] Cho phép spike kiểm chứng OpenHands Agent Server trước khi chốt D-006.
