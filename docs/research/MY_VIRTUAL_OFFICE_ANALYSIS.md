# Phân tích repo `my-virtual-office` (eliautobot) — Research Agent 03

- **Repo tham chiếu:** `D:\Code\Tuyen\Agent management\my-virtual-office` (clone của https://github.com/eliautobot/my-virtual-office)
- **Commit phân tích:** `f875345` (2026-08-15) — "docs: clarify v0.7.4 package availability", version 0.7.4
- **License:** `AGPL-3.0-or-later` (file `LICENSE`, 35 KB — bản GNU AGPL v3 đầy đủ; README mục "License" xác nhận và nhắc "Commercial licensing is available separately")
- **Mục tiêu tài liệu:** Mô tả kiến trúc, cách mô hình hóa agent/công ty, orchestration, provider integration, state, sandbox, office rendering, skills, human review; đánh giá license và kết luận phần nào chỉ được tham khảo ý tưởng, phần nào AI WORKSPACE phải viết mới.

> **Kết luận license (đọc trước):** Toàn bộ code của repo này là AGPL-3.0. **KHÔNG được copy code (Python, JS, CSS, HTML, JSON layout, prompt template nguyên văn) vào AI WORKSPACE** nếu sản phẩm của chúng ta là proprietary hoặc MIT. Chỉ được dùng repo này làm **tài liệu tham khảo kiến trúc / ý tưởng**, rồi tự viết lại bằng TypeScript/NestJS/React theo thiết kế riêng. Chi tiết ở mục 10.

---

## 1. Kiến trúc tổng thể và tech stack

### 1.1 Tổng quan

My Virtual Office (MVO) là một **"control surface + visualizer"** self-hosted cho các agent harness bên ngoài. Nó **không tự chạy LLM**; nó kết nối tới các runtime có sẵn (OpenClaw gateway, Hermes API, Codex CLI, Claude Code CLI, OpenCode, Antigravity), đọc trạng thái/sự kiện của chúng rồi vẽ thành văn phòng 2D trong trình duyệt.

```
Browser (index.html + game.js + chat.js + projects.js ...)
        │  HTTP polling (/status 5s, /agent-chat 3s, /api/presence ...)
        │  WebSocket proxy (port 8091) → OpenClaw gateway
        ▼
server.py  (Python 3.12, http.server.ThreadingHTTPServer, ~22k dòng, 1 file)
   ├── providers/registry.py      Provider contract v1.1 + manifest + capability map
   ├── providers/builtin.py       build_provider_registry(): OpenClaw, Hermes, Codex, Claude Code, OpenCode, Antigravity
   ├── providers/claude_code.py   subprocess `claude -p --output-format stream-json`
   ├── providers/codex.py         JSON-RPC client `codex app-server --stdio`
   ├── providers/hermes.py        HTTP/SSE client tới Hermes API server
   ├── gateway_presence.py        Presence coordinator (in-memory, lifecycle_* API)
   ├── project_store.py           Markdown + YAML-frontmatter project/task store
   ├── layout_library.py          Layout marketplace (.mvo-layout.json)
   ├── license.py                 Lemon Squeezy license check, demo limits
   └── discovery.py               Roster discovery từ ~/.openclaw, ~/.codex, ~/.claude
        │
        ▼  filesystem (VO_STATUS_DIR=/data, Docker volume vo-data)
   presence snapshot, projects/*.md, meetings, scores, skills-library/, layouts/, *.jsonl
```

### 1.2 Ngôn ngữ, framework

| Lớp | Công nghệ | Ghi chú |
|---|---|---|
| Backend | Python 3.12 thuần, `http.server`, `websockets==17.0.1` | **Không** dùng Flask/FastAPI; `app/Dockerfile` chỉ `pip install websockets`. `server.py` là God-file 22.229 dòng với `do_GET/do_POST/do_PUT/do_DELETE` khổng lồ. |
| Frontend | Vanilla JS (ES2015+), HTML5 Canvas 2D, CSS thuần | Không React/Vue/Phaser/PixiJS. `game.js` 17.334 dòng, `chat.js` 5.506 dòng, `projects.js` 2.315 dòng, `style.css` 5.011 dòng. Một số module `.mjs` (preview hub, browser window controller). |
| Markdown | `marked.min.js` | Render chat markdown. |
| Persistence | File JSON/JSONL/Markdown trên volume | **Không có database**; không PostgreSQL/SQLite (SQLite chỉ dùng để *đọc* auth store của OpenClaw). |
| Container | `python:3.12-slim`, `entrypoint.sh` chạy `python3 server.py` | Multi-arch amd64/arm64 qua GitHub Actions (`.github/workflows/docker-publish.yml`). |
| Website | `website/` static HTML + nginx:alpine | Landing page bán license ($9.99), không liên quan runtime. |

### 1.3 Services trong `docker-compose.yml`

```yaml
services:
  virtual-office:        # image ghcr.io/eliautobot/my-virtual-office:latest, ports 8090 (HTTP) + 8091 (WS proxy)
    volumes:
      - ${VO_OPENCLAW_PATH:-~/.openclaw}:/openclaw        # đọc/ghi OpenClaw home
      - vo-data:/data                                     # mọi state của office
      - ${VO_HERMES_HOST_DATA:-~/.hermes}:/data/hermes:rw # chỉ profile/skill resources
    environment: [~80 biến VO_* cho OpenClaw, Hermes, Codex, Claude Code, OpenCode, Antigravity]
    healthcheck: GET /health
  hermes:                # profile "hermes", image nousresearch/hermes-agent:v2026.8.3, network internal-only, expose 8642
  # agent-browser (commented): kasmweb/chromium:1.17.0, ports 6901 (KasmVNC) + 9222 (CDP)
networks:
  hermes-internal: { internal: true }
```

Điểm đáng chú ý: CLI của Codex/Claude Code **phải được cài vào image hoặc mount vào container**; MVO không tự cài. `CODEX_HOME` và `CLAUDE_CONFIG_DIR` được trỏ vào `/data/codex-home`, `/data/claude-code-home` để auth nằm trong volume.

### 1.4 Entry/bootstrap

`app/entrypoint.sh`:
```sh
SHARED="${VO_STATUS_DIR:-/data}"; mkdir -p "$SHARED"; chmod 777 "$SHARED"
CACHE_V=$(date +%s); sed -i "s/?v=[0-9]*/?v=${CACHE_V}/g" index.html   # cache-busting thủ công
exec python3 server.py
```

`server.py` khởi động: `_load_vo_config()` (merge `vo-config.json` + env), `build_provider_registry()`, `gateway_presence.start()` (thread WebSocket observer tới OpenClaw), `start_ws_server()` (proxy 8091), `start_http_server()` (8090), `_wf_auto_resume_on_startup()` (resume workflow pipeline từ `workflow-state.json`).

---

## 2. Cách định nghĩa AI agent / employee, role, mô hình công ty

### 2.1 Agent = "profile" của một provider, không có bảng Employee riêng

MVO không có schema Employee. Mỗi agent là một **row được discover** từ provider, rồi registry chuẩn hóa thành canonical fields (`app/providers/registry.py::_normalize_agent`):

```python
result.update({
    "id": agent_id,                       # ví dụ "claude-code-frontend-dev"
    "statusKey": status_key,              # key dùng trong /status presence map
    "providerKind": provider_id,          # openclaw | hermes | codex | claude-code | opencode | antigravity
    "providerType": ... or manifest.provider_type,   # "runtime" | "harness" | "api"
    "providerAgentId": native_id,
    "profile": row.get("profile") or native_id,
    "name": ..., "emoji": ..., "role": row.get("role") or manifest.name,
    "capabilities": capabilities,         # map 22 capability flags
    "providerConnectionId": ...,          # ownership = providerKind + connectionId + providerAgentId
    "available": bool(row.get("available", True)),
    "connectionState": ...,
})
```

Capability list (`CAPABILITY_NAMES`): `discover, health, chat, streaming, sessions, sessionCreate, sessionDelete, sessionSwitch, interrupt, approvals, agentCreate, agentDelete, profileEdit, resourcesRead, resourcesWrite, skills, models, projects, meetings, agentToAgent, attachments, tools`. UI ẩn tính năng theo flag này — đây là ý tưởng tốt để tái dùng (viết mới).

### 2.2 Persona/role được materialize thành file trong workspace

Khi tạo agent, server ghi các file Markdown "nhân cách" vào workspace của agent. OpenClaw (`server.py::_agent_template_files`):

```python
return {
    "IDENTITY.md": "- **Name:** {name}\n- **Creature:** {role} — {agent_kind} agent\n- **Emoji:** {emoji}",
    "SOUL.md":     "You are **{name}** {emoji} — {role}. ## Style ...",
    "USER.md":     "...",
    "AGENTS.md":   "## Role\n{role}\n## Core Rules ... ## Communication ... ## Memory\n- Daily logs: memory/YYYY-MM-DD.md",
    "HEARTBEAT.md": "...", "MEMORY.md": "...", "TOOLS.md": "...",
}
```

Claude Code (`app/providers/claude_code.py::create_agent`): ghi `office-agent.json` (meta), `IDENTITY.md`, `AGENTS.md`, `CLAUDE.md`, `.claude/agents/<profile>.md` (subagent frontmatter `name/description/model`), và tùy chọn `$CLAUDE_CONFIG_DIR/agents/<profile>.md`. Codex: `AGENTS.md` + `.codex/agents/<profile>.toml` + `config.toml` (`sandbox_mode`, `approval_policy`).

`docs/UNIVERSAL-AGENT-HARNESS-SPEC.md` §6.7 đề xuất mô hình canonical `OfficePromptPack { identity, mission, style, boundaries, memory, userContext, projectContext }` và coi `AGENTS.md/SOUL.md/CLAUDE.md` chỉ là "materialization" — **chưa implement**, chỉ là spec. Đây là ý tưởng rất phù hợp cho AI WORKSPACE (lưu PromptPack trong PostgreSQL, render ra `CLAUDE.md`/`AGENTS.md` cho từng CLI).

### 2.3 Mô hình "công ty"

- **Branch** = department. Định nghĩa phía client trong `game.js`:
  ```js
  const DEFAULT_BRANCHES = [ { id: 'ENG', name: 'Engineering', emoji: '⚙️', theme: 'branch-blue' }, ... ];
  const UNASSIGNED_BRANCH = { id: 'UNASSIGNED', name: 'Unassigned', emoji: '❓', theme: 'branch-gray' };
  ```
  Branch chỉ dùng để gán màu tường/khu vực desk, không có quyền hạn hay reporting line.
- **Boss desk** (`deskType === 'boss'`) chỉ là vị trí ngồi đặc biệt; không có khái niệm CEO/PM/manager trong backend.
- **Không có org chart, không có role hierarchy, không có "hire/fire" lifecycle** ngoài `POST /api/agent/create` / `DELETE /api/agent/delete`.
- Agent được giới hạn 3 trong demo mode (`license.py::get_agent_limit`, `_apply_agent_limit_balanced`).

**Kết luận mục 2:** MVO mô hình hóa agent rất mỏng (profile + role string + emoji + branch). AI WORKSPACE cần schema đầy đủ hơn (Employee, Role, Team, Manager relation, Skill set, Cost center) — phải thiết kế mới hoàn toàn.

---

## 3. Orchestration: task assignment, workflow, PM-like role, communication, meetings

### 3.1 Projects & Kanban (file-based)

`app/project_store.py::MarkdownProjectStore` lưu mỗi project thành thư mục `STATUS_DIR/projects/<slug>--<id>/` với `project.md` và `tasks/<title-slug>--<id8>.md` (YAML frontmatter + sections `## Description / ## Comments / ## Attachments / ## Review Check`). Cột mặc định: Backlog → To Do → In Progress → Review → Done (`docs/design-history/PROJECTS-FEATURE-SPEC.md`). Task có `assignee` (agent key), `checklist[]`, `priority`, `tags`, `reviewCheck[]`.

### 3.2 Workflow pipeline — "PM" là một vòng lặp Python, không phải agent

Không có agent PM. "Project manager" là thread `_wf_run_pipeline_inner(project_id, single_task, wf, stop_flag)` trong `server.py` (dòng 15098–15495). Luồng:

1. Lấy task kế tiếp ở Backlog (`_wf_next_backlog_task`); nếu còn task đang In Progress/Review thì **block**, không lấy task mới (`phase = blocked_by_active_task`).
2. Task phải có `assignee`, nếu không → `error: "Please assign an agent to all tasks"`. **Không có auto-assignment.**
3. Move → In Progress, ghi task file, dọn session cũ (`_wf_cleanup_task_sessions`), gọi agent với `_wf_build_task_prompt()`.
4. Trích tool activity từ session JSONL (`_wf_extract_session_activity`: files_read/edited/written, exec_commands, browser_actions, tool_call_count).
5. Move → Review; vòng lặp tối đa `max_review_cycles = 5`: gọi **cùng agent** với `_wf_build_review_prompt()` (self-review), parse `REVIEW_ITEM_N: PASS|NEEDS_MORE_WORK|DID_NOT_PASS|REQUIRES_USER_REVIEW`.
6. **Tool-call verification**: nếu agent claim all-PASS nhưng không có read/exec/browser call → reject:
   ```python
   review_verified = review_has_reads or review_has_exec or review_has_browser
   review_visual_verified = review_has_browser if task_needs_visual_review else True
   if all_pass and (not review_verified or not review_visual_verified):
       ... all_pass = False; failed_items = review_results
   ```
7. Fail → rework prompt, move về In Progress, quay lại Review. Safety cap: 3 lần parse fail hoặc 3 lần rework liên tiếp → `awaiting_human_intervention`.
8. `REQUIRES_USER_REVIEW` → `awaiting_user_review`, pipeline **poll 5s** chờ user sửa `reviewCheck` qua `PUT /api/projects/{id}/tasks/{taskId}/review-check`.
9. Done → mark checklist done, `_award_points` (gamification `scores.json`), cleanup session; `autoMode` quyết định tiếp tục task kế hay dừng.

Prompt task (trích `_wf_build_task_prompt`):
```
You have been assigned a task. Complete it fully on your own. Do NOT ask for clarification...
MANDATORY RULES — VIOLATIONS WILL FAIL REVIEW:
1. You MUST use tools (read, edit, exec, browser) to make REAL changes to actual files. Text-only responses WILL BE REJECTED.
...
A reviewer will independently verify your work by reading the actual files and browsing the app.
```

`_wf_call_agent` giữ **một session key cho mỗi (agent, project, task)** để work → review → rework dùng chung context và tận dụng prompt caching; với provider non-OpenClaw thì route qua `_handle_provider_chat`.

### 3.3 Agent-to-agent communication

`docs/AGENT_PLATFORM_COMMUNICATIONS.md` + `server.py::_handle_agent_platform_comm_send`:
- `POST /api/agent-platform-communications/send {fromAgentId, toAgentId, message, conversationId}` → server gọi agent đích (synchronous), lưu request + reply vào `VO_STATUS_DIR/agent-platform-communications.jsonl`.
- Lịch sử được merge vào `/agent-chat` để vẽ speech bubble cho cả hai agent.
- Agent học cách dùng qua built-in skill `AgentPlatform-to-AgentPlatform_Communications` (SKILL.md được seed vào Skills Library và expose tại `GET /api/agent-platform-communications/skill`).
- Đây là **request/reply đồng bộ qua HTTP**, không có message bus, không có thread/topic model thực sự (spec §11 của UNIVERSAL-AGENT-HARNESS-SPEC đề xuất `OfficeThread/OfficeMessage` với broker nhưng chưa làm).

### 3.4 Meetings

`server.py::_handle_meeting_create`:
```python
meeting = {"id", "topic", "purpose", "kind": "discussion", "type": "1on1"|"group",
           "organizer", "status": "active", "participants": [...],
           "rules": {"mode": "discussion-not-work", "endWhen": "purpose-complete",
                     "resumeStateAfterEnd": "working-or-idle"}}
```
Meeting lưu vào file status (`_meetings` list), `gateway_presence.set_meetings()`; `POST /api/meetings/end` **bắt buộc summary** từ organizer, lưu vào history. Frontend `processMeetings()` đưa agent tới bàn họp (10 ghế) hoặc `visitAgent()` cho 1-on-1. Meeting không tự động chạy hội thoại LLM giữa các agent — chỉ là trạng thái + visual; nội dung do agent/user gọi API.

### 3.5 Automations

`cron.html` là UI cho OpenClaw cron (provider-native). Spec §15 đề xuất "office-runtime executor" cho Claude Code/Codex nhưng chưa có.

**Kết luận mục 3:** Ý tưởng đáng tái hiện: (a) pipeline Backlog→InProgress→Review→Done với self-review + tool-call verification + rework cap + human escalation; (b) 1 session/task; (c) meeting phải kết thúc bằng summary; (d) comm log visible. Thiếu: PM agent thật, auto-assignment, parallel tasks (pipeline tuần tự 1 task/project), dependency graph, message bus.

---

## 4. AI provider integration (LLM/CLI, auth, Claude Code / Codex / OpenHands)

### 4.1 Provider contract (Provider SDK v1.1)

`docs/PROVIDER-SDK.md` + `providers/registry.py`: mỗi provider có `ProviderManifest(id, name, provider_type, capabilities, connection_schema, creation_schema, settings_schema, resource_schema, skill_schema)` và implement các method theo capability (`test`, `discover_agents`, `send_chat_message`, `list_sessions`, `interrupt`, `create_agent`, `delete_agent`, `update_profile`, `pending_approval`, `respond_approval`). Extension loader: `vo_provider_*.py` trong `/data/provider-extensions` export `build_provider(context)`. `GET /api/providers/conformance` kiểm tra contract.

### 4.2 Claude Code — subprocess stream-json

`app/providers/claude_code.py::send_chat_message`:
```python
cmd = [self.binary or "claude", "-p", "--output-format", "stream-json", "--verbose", "--include-partial-messages"]
if session_id: cmd.extend(["--resume", session_id])
if safe_profile != "main": cmd.extend(["--agent", safe_profile])
if self.model: cmd.extend(["--model", self.model])
if permission_mode == "bypassPermissions": cmd.append("--dangerously-skip-permissions")
elif permission_mode: cmd.extend(["--permission-mode", permission_mode])
cmd.append(message)
proc = subprocess.Popen(cmd, cwd=workspace, env=self._subprocess_env(), stdin=DEVNULL, stdout=PIPE, stderr=PIPE, text=True, bufsize=1)
for raw in proc.stdout:
    item = json.loads(line); state.ingest(item); on_progress(state.as_progress())
```
`_ClaudeStreamState.ingest` xử lý `system` (session_id, model), `assistant/user` message (tool_use/tool_result blocks), `stream_event` (text delta), `result` (usage, is_error). `interrupt()` = `proc.terminate()`. Mỗi lời nhắn = **một process mới** (`claude -p`), tiếp nối qua `--resume <session_id>`. Default `permissionMode = acceptEdits` (`vo-config.json`), env `CLAUDE_CONFIG_DIR` trỏ vào volume. Không dùng Agent SDK, không dùng hooks, không dùng MCP.

### 4.3 Codex — app-server JSON-RPC

`app/providers/codex.py::CodexAppServerClient` spawn `codex app-server --stdio`, gọi `initialize` → `thread/start` hoặc `thread/resume` → `turn/start {approvalPolicy, sandbox}`; đọc notification stream; `interrupt`; server-request approval (`_is_approval_request`) được giữ trong `threading.Condition` chờ `respond_approval(approval_id, choice)`. Fallback `codex exec --json --skip-git-repo-check -C <dir> --sandbox <mode>`. Docker default `VO_CODEX_SANDBOX=danger-full-access` vì bubblewrap không chạy được trong container không privileged.

### 4.4 Hermes, OpenClaw, OpenCode, Antigravity

- **OpenClaw**: WebSocket gateway (protocol v4) qua `openclaw_gateway_client.py`; trình duyệt gọi RPC qua same-origin proxy `/api/openclaw/rpc` + SSE `/api/openclaw/events`; MVO còn **đọc/ghi auth store SQLite/JSON của OpenClaw** để quản lý API key/model (`_read_openclaw_auth_sqlite`, `_save_openclaw_api_key`).
- **Hermes**: HTTP `/v1/runs` + SSE, hydrate `conversation_history` thủ công (giới hạn 120 msg/120k chars); thêm mode "platform plugin" (`integrations/hermes-platform/my_virtual_office/adapter.py`, subclass `BasePlatformAdapter`) để Hermes **poll** MVO như một messaging platform (`/api/hermes-platform/poll|ack|reply|heartbeat`, Bearer token).
- **OpenCode / Antigravity**: adapter CLI tương tự (`providers/opencode.py`, `providers/antigravity.py`), disabled mặc định.
- **OpenHands: không có**. Không có tích hợp OpenHands ở bất kỳ đâu trong repo.

### 4.5 Auth

README: "CLI authentication remains native to each framework... Virtual Office does not copy or expose provider credentials." Thực tế: auth Claude/Codex nằm trong `CLAUDE_CONFIG_DIR`/`CODEX_HOME` được mount; Hermes dùng `API_SERVER_KEY`; OpenClaw dùng gateway token lấy từ `openclaw.json`. Secrets được redact khi trả về UI (`_redact_provider_secrets`, `_mask_secret`). Không có user auth cho chính MVO (khuyên dùng Tailscale).

**Kết luận mục 4:** Cách gọi `claude -p --output-format stream-json --resume` và `codex app-server` JSON-RPC là **pattern** mà AI WORKSPACE có thể dùng (tự viết TypeScript adapter với `child_process.spawn`). Phần Codex approval bridging (server-request → UI card → respond) là ý tưởng hữu ích.

---

## 5. Session & state management, persistence

### 5.1 Không có DB — tất cả là file trong `VO_STATUS_DIR`

| Dữ liệu | Vị trí | Ghi chú |
|---|---|---|
| Config | `vo-config.json` (+ env override) | `_load_vo_config` |
| Presence snapshot | `VO_STATUS_DIR/...` qua `gateway_presence.save_snapshot/load_snapshot` | in-memory `_state: agent_id → {state, task, updated, source}` |
| Office layout | `office-config.json` + localStorage key `vo-product-office-config` | `game.js::saveOfficeConfig` |
| Projects/tasks | `projects/<slug>/project.md`, `tasks/*.md` | frontmatter; legacy `projects.json` migrate |
| Workflow state | `workflow-state.json` | `_wf_persist_state`, auto-resume |
| Meetings, scores | file status JSON, `scores.json` | |
| Provider chat history | `<provider>-history-<profile>.json`, session mirror files | `_save_chat_session_mirror`, `_load_provider_history` |
| Active session | `_provider_active_session_path` | theo dõi native thread mới nhất (`_sync_provider_active_session`) |
| A2A comm | `agent-platform-communications.jsonl` | append-only |
| Skills library | `skills-library/<name>/SKILL.md` | |
| Layouts | `layouts/*.json` | |
| Discovery cache | `_save_discovery_cache` | giữ agent "unavailable" thay vì xoá |

### 5.2 Session model

- Session key format `provider:profile:sessionId` (`_provider_session_key`); `@new:` prefix cho session chưa có id native.
- Native session là nguồn sự thật: Claude Code đọc `$CLAUDE_CONFIG_DIR/projects/**/*.jsonl` (`_claude_code_jsonl_to_chat_messages`), Codex `thread/list`, Hermes Sessions API. MVO giữ bản "mirror" để hiển thị nhanh và merge (`_merge_chat_session_messages` theo fingerprint).
- Run model: `_handle_provider_run_start` tạo `run_id = provider-<kind>-<ms>-<uuid8>`, giữ `events[]` + `threading.Condition`; UI long-poll `GET /api/provider-runs/<run_id>` để nhận progress. Lock `_PROVIDER_RUNS_LOCK` tránh 2 cửa sổ chat ghi cùng một native session.
- Presence TTL: manual override 30s, idle timeout 120s, `FINISHING_GRACE_SEC = 12`, stale working → idle sau 45–180s nếu không có active source (`server.py::_normalize_presence_entry`).

### 5.3 Multi-window chat

`docs/MULTI-CHAT-ARCHITECTURE.md`: 1 primary + 3 secondary slot, "shared transport, isolated window state", map `runId → slotId`. Ý tưởng dùng được.

**Kết luận mục 5:** AI WORKSPACE dùng PostgreSQL nên không tái dùng gì ở lớp persistence; nhưng mô hình `Run { runId, events[], done, result }` + long-poll/SSE và "native session là nguồn sự thật, DB chỉ mirror" là thiết kế hợp lý.

---

## 6. Terminal / file execution và sandboxing

### 6.1 Không có sandbox do MVO sở hữu

MVO **không** tự chạy code; việc thực thi do CLI/harness làm:
- Claude Code: `permissionMode` (`acceptEdits` mặc định; `bypassPermissions` → `--dangerously-skip-permissions`), cwd = workspace của agent (`/data/claude-code-agents/<profile>`).
- Codex: `sandbox_mode` (`workspace-write` mặc định, Docker `danger-full-access`), `approval_policy=never`.
- OpenClaw/Hermes: sandbox nằm trong gateway của họ.

Container MVO chạy root-ish (`chmod 777 /data`), mount `~/.openclaw` read-write. **Không có per-task container, không có network policy, không có resource limit** cho agent. `.env.example` ghi rõ: "Docker containers usually cannot run Codex's bubblewrap sandbox unless they are started with extra privileges, so the Docker default is full container access."

### 6.2 File access từ UI ("Agent Desk")

`server.py::_resolve_workspace_file`, `_save_workspace_text_file`: path-scoped theo `resourceSchema` của provider (ví dụ Hermes: `SOUL.md`, `IDENTITY.md`, `workspace/**/*.md`, `skills/**/SKILL.md`; `config.yaml/auth.json/.env` đánh dấu `sensitive, readable: False`). Có preview diff, atomic write, SHA revision conflict, backup/restore (`_resource_backup_root`), reject traversal/symlink. Đây là một lớp an toàn tốt để tham khảo.

### 6.3 Kasm browser (optional)

`kasm-browser-config/`: `kasmweb/chromium:1.17.0` + script `post-boot.sh` cài Google Chrome, `chromium-browser` wrapper chạy Chrome với `--remote-debugging-port=9222 --remote-allow-origins=* --no-sandbox` + stealth flags (`--disable-blink-features=AutomationControlled`), `browser-supervisor.sh` auto-restart, `start-cdp-proxy.sh` socat 9223→9222. MVO panel (`browser-panel.js`) nhúng KasmVNC viewer (6901) để user xem agent lướt web; agent điều khiển qua CDP. Đây là **browser cho agent**, không phải sandbox terminal. `docs/VIRTUAL_OFFICE_AGENT_TOOLS.md` thừa nhận "Provider-neutral browser action endpoint is not implemented yet."

### 6.4 Preview bubbles

`agent-preview-hub.mjs` phát hiện tool call có path/URL (regex `FILE_TOOL_PATTERN`, `BROWSER_TOOL_PATTERN`) và hiện preview (image/markdown/html/pdf/video) neo vào agent; server `_resolve_agent_preview_file` giới hạn trong agent root, chặn hidden/credential/private key.

**Kết luận mục 6:** AI WORKSPACE cần Docker sandbox per-agent/per-task tự thiết kế; MVO không cung cấp mẫu. Kasm+CDP là ý tưởng khả thi cho "agent browser" nhưng cấu hình `--no-sandbox` + stealth flags không nên copy.

---

## 7. 2D office rendering

### 7.1 Engine

- **Không dùng game engine**; Canvas 2D API thuần trong `game.js`. `TILE = 40` px, `DPR = devicePixelRatio || 2`, `requestAnimationFrame(loop)`; README nói "100 FPS rendering".
- Camera: zoom 0.5–3.0 (`CAM_ZOOM_MIN/MAX`), `screenToWorld`, `clampCamera`, `applyCameraTransform`.
- Environment: `drawEnvironment()` vẽ sàn (2 màu ca-rô), top wall sections theo branch màu, interior walls (`drawInteriorWalls`, 6px body + trim + shadow, z-order occluder `drawInteriorWallOccluders`), ambient day/night overlay (`_updateAmbientCache`, `drawAmbientOverlay`, tính giờ mặt trời `_calcSunTimes`), cửa sổ có thời tiết thật (`pollWeather` 10 phút, `drawWeatherOnWindow` vẽ mây/sét procedurally), rim light theo đèn (`getRimLight`).

### 7.2 Characters — procedural pixel-art, không sprite sheet

Toàn bộ nhân vật được vẽ bằng `fillRect/ellipse` theo config appearance (`Agent.draw()` ~ dòng 3793–4340, helper `_drawHairByConfig`, `_drawCostume`, `_drawHeadwear`, `_drawGlasses`, `_drawHeldItem`, `_drawFacialHair`). `grep drawImage` chỉ thấy costume cache canvas và ảnh chat. Appearance: skin, hair style/color, eye, eyebrow, facial hair, glasses, headwear, costume (lobster suit, cape), held item, gender. Animation: `walkBob = |sin(tick*0.2)|*4`, `legOffset`, `breathe`, blink timer, talk timer, sit offset 8px, yawn/scratch head idle pose.

Pet: `class OfficePet` (cat/pug/lobster) với state machine riêng (`sitting, walking, walking_to_sleep, walking_to_greet, walking_to_curious, being_pet, chased`), có directional sprites front/back/side (vẫn procedural).

### 7.3 Movement, desks, rooms

- Desk auto-assign: `_autoAssignDesks(agentDefs)` gán agent vào furniture `desk` chưa có chủ; `LOCATIONS` chứa bossDesk, meeting table, lounge, cooler...
- Pathfinding: `buildCollisionGrid()` đánh dấu edge-blocking per tile (`{top,right,bottom,left}`) từ `officeConfig.walls.interior` (toạ độ tile, tường nằm *giữa* các tile); `findPath()` A* 4 hướng, Manhattan heuristic, open list tuyến tính, `maxIter = cols*rows*2`; chỉ tính lại khi `targetX/Y` đổi; fallback đi thẳng nếu không có tường.
- Collision agent–agent: steering đẩy mềm (`COLLISION_RADIUS = 22`, `COLLISION_PUSH = 1.4`), `isSpotOccupied/findOpenSpot`, wall clearance 14px.
- Object service queue: hàng chờ cho máy pha cà phê/máy bán hàng (`enqueueAgentForObjectService`, `getObjectQueueWaitPosition`).
- Social: paper airplane, rock-paper-scissors, darts, ping-pong, gatherings (`startGathering`, tránh tường), pet interaction.
- Furniture catalog `CATALOG_CATEGORIES` 25+ item với `FURNITURE_BOUNDS`; editor drag-drop snap grid, undo stack 30, group select → save layout, text label, interactive window.

### 7.4 Mapping hoạt động thật → visual

Chuỗi: provider events → `gateway_presence.lifecycle_*` → `_state[agent]` → `GET /status` (poll 5s) → `pollStatus()` → `agent.moveTo(newState)`.

```js
// game.js::pollStatus
const entry = data[agent.statusKey];
if (!(agent.meetingId || agent.state === 'visiting')) {
  const newState = entry.state || 'idle';
  if (newState !== agent.state) agent.moveTo(newState);   // working|idle → về desk; meeting → bàn họp; break → cooler; lounge → sofa
}
if (entry.thought) ...  // thought bubble 💭  (từ tool activity)
if (entry.speech)  ...  // speech bubble 💬  (reply cuối / A2A message)
if (entry.notify)  ...  // đèn báo có response
```

Presence vocabulary server (`_normalize_presence_entry`) gom ~25 alias (`thinking, tool_start, running_command, streaming...`) về 6 state: `working, finishing, idle, meeting, break, offline`. `gateway_presence._format_tool_task(name, args)` tạo text "Reading file.py" / "Running: npm test" làm `task` hiển thị dưới tên agent. Màn hình desk sáng khi `working|finishing` (`_isDeskScreenActive`). `pollAgentChat()` (3s) kéo `/agent-chat` để vẽ chat bubble với tool lines (`formatAgentChatToolLine`).

**Kết luận mục 7:** Cách tiếp cận "state → location" (`moveTo`) + A* trên grid 40px + presence normalizer là ý tưởng cốt lõi để tái hiện. AI WORKSPACE nên dùng PixiJS/Phaser + sprite sheet (asset riêng, license rõ ràng) thay vì vẽ thủ công 17k dòng; dùng WebSocket push thay vì polling 5s.

---

## 8. Skills / plugins

- **Skills Library** (`docs/SKILLS-LIBRARY-SPEC.md`, `server.py::_handle_skills_library_*`): thư mục `STATUS_DIR/skills-library/<name>/SKILL.md` với frontmatter `name/description` (format AgentSkills); API list/get/create/delete/upload/apply/save-from-agent. "Apply" = **copy** SKILL.md vào `workspace/skills/<name>/` (hoặc `.claude/skills/`) của agent; agent sở hữu bản copy độc lập.
- **Built-in skills** được seed (`_builtin_office_skill_contents`): `AgentPlatform-to-AgentPlatform_Communications`, `VirtualOffice-Presence-and-Status`, `VirtualOffice-Browser-Control`, `VirtualOffice-Meetings`, `VirtualOffice-Projects-and-Tasks` — chúng dạy agent gọi REST API của office (ví dụ `POST /api/presence/<agentId>` để tự set `working|idle|break|meeting`).
- **Skill Workshop** (`_skill_workshop_*`): bridge tới OpenClaw CLI để agent đề xuất skill.
- **Provider extensions**: `vo_provider_*.py` plugin (mục 4.1).
- **Hermes platform plugin** (`integrations/hermes-platform/`): plugin cho Hermes, không phải cho MVO.
- Không có MCP server, không có hook system, không có plugin marketplace.

**Kết luận mục 8:** Ý tưởng "office tự seed skill dạy agent cách dùng API của office" là hay và nên tái hiện (AI WORKSPACE có thể expose MCP server thay vì REST + SKILL.md).

---

## 9. Human approval / review flows

### 9.1 Review parser (`test_review_parser.py` ở root và `tests/test_review_parser.py`)

File root là script tự chứa (copy nguyên hàm `_wf_parse_review_response` và `_wf_review_had_structured_match` "must match server.py exactly") với 10 nhóm test. Logic parser:

1. Quét từng dòng; dòng là "review line" nếu chứa `review_item`, bắt đầu bằng số (`^\d+[\.\):\s]`), hoặc chứa `item `. Ghép status theo thứ tự ưu tiên `requires_user_review > needs_more_work > did_not_pass > pass`, gán tuần tự vào checklist (đánh dấu `_parsed=True`).
2. Nếu không match dòng nào → **freeform fallback**: đếm keyword tích cực (`"lgtm", "ship it", "all items verified"...`) và tiêu cực (`"failed", "missing", "broken"...`); không có tiêu cực → all-pass với `_fallback = freeform_positive_sentiment | freeform_no_negatives`.
3. `review_cycle >= 3` và mọi checklist item `done` → all-pass `_fallback = cycle_3_checklist_done`.
4. Còn lại → `needs_more_work` với `_default=True`.

Các marker `_parsed/_fallback/_default` cho phép pipeline phát hiện "agent trả lời tự do liên tục" → safety cap → escalate người.

### 9.2 Các điểm cần con người

| Tình huống | Phase | Cách giải quyết |
|---|---|---|
| Agent trả `REQUIRES_USER_REVIEW` | `awaiting_user_review` | User sửa status trong UI → `PUT .../review-check`; pipeline poll 5s |
| 5 review cycle thất bại / 3 rework liên tiếp / 3 parse fail | `awaiting_human_intervention` | User kéo task sang Done hoặc Backlog; pipeline poll cột |
| Codex/Hermes approval (exec command, file change) | presence `waiting_approval` | Chat card Approve/Cancel → `POST /api/codex/approval/respond` hoặc `/api/hermes/approval/respond`; `_wait_for_approval_response` block run |
| Claude Code permission | — | Không có bridge; dựa vào `permissionMode=acceptEdits`, hoặc bypass |
| Meeting end | — | Bắt buộc `summary` |

`tests/test_workflow_e2e.py` chạy pipeline thật chống server port 8090 (Backlog→Done, stop, auto mode, rework). `tests/test_e2e_task1_dispatch_*.py` + `app/e2e-task1-dispatch-*.html` là artifact của các lần agent tự chạy e2e (ghi log HTML).

**Kết luận mục 9:** Cần tái hiện: structured review protocol + fallback parser + marker-based safety cap + approval bridge dạng card. Nên cải tiến: reviewer độc lập (agent khác), review theo diff/test result thay vì tự khai, dùng JSON schema output thay vì regex dòng.

---

## 10. AGPL-3.0: hệ quả pháp lý và những gì được phép

### 10.1 License thực tế của repo

- `LICENSE` = GNU Affero General Public License v3 (19 Nov 2007), toàn văn.
- README: "GNU Affero General Public License v3.0 or later (AGPL-3.0-or-later)... If you modify the app and make it available to users over a network, you must offer those users the corresponding source code for your modified version. Paid license keys unlock the hosted/product feature set... Commercial licensing is available separately."
- Không có file CLA/exception; không có dual-license công khai trong repo (chỉ nhắc "available separately").

### 10.2 Điều AGPL-3.0 yêu cầu

1. **Copyleft mạnh (§5, §13):** bất kỳ "work based on the Program" (copy, sửa, dịch sang ngôn ngữ khác, kết hợp vào sản phẩm) phải phát hành toàn bộ mã nguồn tương ứng dưới AGPL-3.0 khi phân phối **hoặc khi cho người dùng tương tác qua mạng** (SaaS). AI WORKSPACE là web app → điều §13 "Remote Network Interaction" áp dụng đầy đủ.
2. **Không tương thích với proprietary / MIT closed-source:** nếu copy một hàm (ví dụ `_wf_parse_review_response`, `findPath`, `Agent.draw`) hoặc dịch nó sang TypeScript gần như 1:1, phần code đó và **toàn bộ codebase liên kết** phải mở nguồn AGPL. MIT có thể "nâng" lên AGPL nhưng không thể ngược lại; sản phẩm thương mại đóng sẽ vi phạm.
3. **Derivative work bao gồm cả dịch ngôn ngữ:** AGPL §0 định nghĩa "modify" bao gồm "to copy from or adapt all or part of the work... other than the making of an exact copy". Port Python→TypeScript vẫn là derivative.
4. **Asset và prompt:** prompt template trong `server.py` (task/review/rework prompt), `default-office-config.json`, CSS theme, SKILL.md built-in đều là một phần của Program → cùng AGPL.
5. **Ý tưởng, thuật toán, API shape, kiến trúc không được bảo hộ bởi bản quyền** (copyright chỉ bảo hộ biểu đạt). A* pathfinding, mô hình state machine, "Backlog→Review→Done với self-review", capability flags, format `REVIEW_ITEM_N: PASS` là ý tưởng/giao thức — được tự thiết kế lại miễn là **viết mới từ mô tả**, không nhìn-chép code.

### 10.3 Kết luận pháp lý (rõ ràng)

- **KHÔNG được** copy/paste, dịch, hoặc "refactor nhẹ" bất kỳ file nào trong `my-virtual-office` vào AI WORKSPACE nếu AI WORKSPACE không phát hành dưới AGPL-3.0. Làm vậy sẽ **"contaminate"** toàn bộ sản phẩm với nghĩa vụ AGPL (phải công bố mã nguồn cho mọi người dùng mạng).
- **KHÔNG được** dùng `default-office-config.json`, `screenshot.png`, `favicon`, CSS theme, prompt text nguyên văn.
- **ĐƯỢC** đọc để hiểu kiến trúc, viết tài liệu phân tích (như file này), lập danh sách "ý tưởng cần tái hiện", rồi **clean-room implement** bằng TypeScript/NestJS/React theo spec riêng. Khuyến nghị quy trình clean-room: người viết spec (đọc MVO) ≠ người viết code (chỉ đọc spec), hoặc tối thiểu không mở file MVO khi code.
- **ĐƯỢC** chạy MVO như một **ứng dụng riêng biệt** (process/container độc lập) để demo/so sánh; AGPL không lan sang khi chỉ chạy cạnh nhau và không link/import. Nếu AI WORKSPACE gọi MVO qua HTTP API như một dịch vụ bên ngoài không sửa đổi, cũng không bị contaminate — nhưng không khuyến nghị vì MVO không có user auth và mô hình dữ liệu quá khác.
- Nếu muốn dùng code thật: phải mua **commercial license** từ tác giả (README nhắc có), hoặc chấp nhận phát hành AI WORKSPACE dưới AGPL-3.0.

---

## 11. Reference-only vs phải viết mới

### 11.1 Ý tưởng đáng tái hiện (clean-room)

1. **Provider manifest + capability flags** điều khiển UI (22 flag) — thiết kế lại thành `ProviderAdapter` interface TypeScript với `capabilities: Set<Capability>`.
2. **CLI adapter pattern**: spawn `claude -p --output-format stream-json --include-partial-messages --resume <id>` và `codex app-server --stdio` JSON-RPC; normalize thành `RunEvent {textDelta, toolStart, toolResult, approvalRequested, usage, done}`.
3. **Presence coordinator** với lifecycle vocabulary (`run_started, output_streaming, tool_started/finished, approval_requested/resolved, run_finished`), TTL/grace, alias normalizer → 6 visual state.
4. **State → location mapping** trong office (`working/idle → desk`, `meeting → table slot`, `break → kitchen`, `waiting_approval → raise hand`), grid-based A*, soft collision steering, object queue.
5. **Workflow pipeline** Backlog→InProgress→Review→Done: 1 session/task, structured review protocol, tool-call verification, rework cap, human escalation phase, auto-resume sau restart.
6. **Approval bridge**: server-request từ CLI → card trong UI → respond; block run bằng Condition/Promise.
7. **Office-seeded skills / MCP tools** dạy agent dùng API của office (presence, meetings, projects, A2A).
8. **A2A comm log** visible dưới dạng speech bubble; meeting bắt buộc summary.
9. **Agent Desk file access** path-scoped theo resource schema, sensitive file ẩn, diff preview, revision conflict, backup.
10. **PromptPack canonical → materialize** `CLAUDE.md`/`AGENTS.md`/subagent `.md` per provider (spec chưa implement trong MVO; ta có thể làm đúng từ đầu).
11. **Layout asset format** (furniture px offset + walls tile offset, kind selection/office) → thiết kế schema JSON riêng.
12. **Multi-window chat**: shared transport, per-slot state, `runId → slot`.

### 11.2 Những gì MVO không có / AI WORKSPACE phải tự thiết kế

- Database schema (PostgreSQL): Company, Team, Employee(Agent), Role, Project, Task, Run, Message, Approval, Meeting, Skill, Cost.
- PM agent thật (planning, decomposition, auto-assignment, dependency, parallel tasks, retries).
- Reviewer độc lập, test-based verification, diff review.
- Docker sandbox per agent/task, network/resource policy, secret injection.
- Auth/multi-user/RBAC cho chính platform.
- Realtime WebSocket push (MVO poll 3–5s).
- Rendering engine (PixiJS/Phaser) + sprite assets có license rõ ràng.
- Message bus / thread model cho A2A (MVO chỉ request-reply đồng bộ).
- Cost/usage tracking từ stream-json `usage` (MVO chỉ đọc `openclaw status --usage`).

### 11.3 Bảng tổng hợp

| Module | Path | Tái sử dụng trực tiếp | Cần chỉnh sửa | Viết mới | Ghi chú |
|---|---|---|---|---|---|
| HTTP server + routing | `app/server.py` (22k dòng) | Không (AGPL) | Không | **Có** | NestJS controllers/modules; tham khảo danh sách ~150 endpoint làm checklist tính năng |
| Provider registry / manifest | `app/providers/registry.py` | Không (AGPL) | Không | **Có** | Tái hiện capability-flag design bằng TS interface |
| Claude Code adapter | `app/providers/claude_code.py` | Không (AGPL) | Không | **Có** | Pattern `claude -p stream-json --resume`; viết `ClaudeCodeAdapter` bằng `child_process` |
| Codex adapter | `app/providers/codex.py` | Không (AGPL) | Không | **Có** | Pattern `codex app-server --stdio` JSON-RPC + approval server-request |
| Hermes / OpenClaw / OpenCode / Antigravity | `app/providers/hermes.py`, `builtin.py`, `opencode.py`, `antigravity.py`, `openclaw_gateway_client.py` | Không | Không | Tùy chọn | Ngoài scope AI WORKSPACE (chỉ Claude Code + Codex) |
| Presence coordinator | `app/gateway_presence.py` | Không (AGPL) | Không | **Có** | Tái hiện lifecycle vocabulary + TTL + normalizer |
| Workflow pipeline | `server.py::_wf_*` (13686–15970) | Không (AGPL) | Không | **Có** | Thiết kế lại thành `WorkflowService` state machine, có PM agent + reviewer độc lập |
| Review parser | `test_review_parser.py`, `server.py::_wf_parse_review_response` | Không (AGPL) | Không | **Có** | Dùng JSON structured output + fallback; giữ ý tưởng marker `_parsed/_fallback/_default` |
| Project/task store | `app/project_store.py` | Không (AGPL) | Không | **Có** | PostgreSQL (TypeORM/Prisma); có thể export Markdown cho agent đọc |
| Meetings | `server.py::_handle_meeting_*` | Không | Không | **Có** | Giữ rule "end requires summary", 1on1/group |
| A2A communications | `server.py::_handle_agent_platform_comm_*`, `docs/AGENT_PLATFORM_COMMUNICATIONS.md` | Không | Không | **Có** | Thay bằng message bus + thread; visible bubble |
| Skills library | `server.py::_handle_skills_library_*`, `docs/SKILLS-LIBRARY-SPEC.md` | Không | Không | **Có** | SKILL.md format là chuẩn AgentSkills (mở), nội dung built-in skill của MVO là AGPL → tự viết |
| Agent Desk file access | `server.py::_resolve_workspace_file`, `_save_workspace_text_file` | Không | Không | **Có** | Tái hiện resource schema + backup + conflict |
| Office canvas engine | `app/game.js` (17k dòng) | Không (AGPL) | Không | **Có** | PixiJS/Phaser + sprite sheet; tái hiện `moveTo(state)`, A*, collision, desk auto-assign |
| Pathfinding A* | `game.js::findPath`, `buildCollisionGrid`, `docs/design-history/TASK-walls-pathfinding.md` | Không | Không | **Có** | A* là thuật toán công cộng; viết lại từ mô tả hoặc dùng lib MIT (`pathfinding`, `easystarjs`) |
| Character appearance | `game.js::Agent.draw`, `_drawHair*`, `_drawCostume` | Không | Không | **Có** | Dùng sprite asset có license (CC0/MIT) thay vì vẽ procedural |
| Office pet | `game.js::OfficePet` | Không | Không | Tùy chọn | Tính năng phụ |
| Furniture catalog & editor | `game.js::CATALOG_CATEGORIES`, `FURNITURE_BOUNDS`, `office-layouts.js`, `layout_library.py` | Không | Không | **Có** | Tự định nghĩa schema layout JSON |
| Default office layout | `app/default-office-config.json` | Không (AGPL) | Không | **Có** | Tự vẽ layout mặc định |
| Chat UI | `app/chat.js`, `chat-events.js`, `chat-markdown.js`, `docs/MULTI-CHAT-ARCHITECTURE.md` | Không | Không | **Có** | React components; giữ ý tưởng slot-based multi-window |
| Preview bubbles | `agent-preview-hub.mjs`, `browser-window-controller.mjs` | Không | Không | Tùy chọn | Ý tưởng: detect path/URL trong tool call → preview |
| Projects UI | `app/projects.js`, `projects.css` | Không | Không | **Có** | React Kanban (dnd-kit) |
| Themes/CSS | `style.css`, `ui-themes.css`, `ui-modern.css` | Không (AGPL) | Không | **Có** | Tailwind/own design system |
| License/paywall | `app/license.py` | Không | Không | Không cần | Lemon Squeezy; không liên quan |
| PC metrics / Whisper / SMS | `pc-metrics-server.py`, `whisper-server.py`, `sms-panel.js` | Không | Không | Không cần | Ngoài scope |
| Kasm agent browser | `kasm-browser-config/*`, compose `agent-browser` | Không (AGPL script) | Không | Tùy chọn | Có thể dùng image `kasmweb/chromium` trực tiếp (license Kasm riêng) với script tự viết; cân nhắc Playwright headless thay thế |
| Hermes platform plugin | `integrations/hermes-platform/` | Không | Không | Không cần | Ngoài scope |
| Docker compose / Dockerfile | `docker-compose.yml`, `app/Dockerfile` | Không | Không | **Có** | Stack khác (Node + Postgres + sandbox) |
| CI publish | `.github/workflows/docker-publish.yml` | Không (nhưng chỉ là config chuẩn) | — | **Có** | Viết workflow riêng; cấu trúc build-push multi-arch là pattern chung của GitHub Actions |
| Docs/spec | `docs/UNIVERSAL-AGENT-HARNESS-SPEC.md`, `PROVIDER-SDK.md`, `VIRTUAL_OFFICE_AGENT_TOOLS.md` | Không copy văn bản | — | **Có** | Đọc để lấy ý tưởng mô hình `OfficeAgent/OfficeThread/OfficeProject/OfficeAutomation`; viết spec riêng bằng lời của mình |
| Tests | `tests/*.py`, `*.js`, `*.mjs` | Không | Không | **Có** | Tham khảo phạm vi test (review parser, session durability, approval, concurrency) |
