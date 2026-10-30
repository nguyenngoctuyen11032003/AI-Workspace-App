# OPENHANDS (Agent Canvas) — Phân tích kỹ thuật cho AI WORKSPACE

- Repo phân tích: `D:\Code\Tuyen\Agent management\OpenHands` (clone `OpenHands/OpenHands`, commit `7219c19`, 2026-10-29, MIT)
- Phiên bản: `@openhands/agent-canvas` **1.34.0**; pin `openhands-agent-server` **1.59.0**, `openhands-automation` 1.26.1 (`config/defaults.json`)
- Người phân tích: Agent 04 (OpenHands Research Analyst)

> **Kết luận quan trọng nhất ngay từ đầu:** repo này **KHÔNG còn là "OpenHands" Python agent framework** mà nhiều người biết (AgentController, EventStream, DockerRuntime, CodeActAgent...). Từ commit `cb9138caf` (2026-07-27, "clear repository for Agent Canvas migration") toàn bộ Python đã bị xoá; trước đó `e86067c15` (2026-04) đã "Removed V0 runtime/controller". Repo hiện tại là **Agent Canvas** — một frontend React/TypeScript + bộ launcher Node (bin/scripts/electron/docker) **điều khiển một backend Python riêng** là **Agent Server** trong repo `OpenHands/software-agent-sdk` (cài qua `uvx openhands-agent-server`, không vendored). Mọi logic agent/runtime/sandbox/LLM thực sự nằm ở SDK đó; repo này chỉ chứa **mirror TypeScript của wire contract** (event schema, settings, request payload) và orchestration.

---

## 0. Bản đồ repo (cái gì nằm ở đâu)

| Thư mục | Nội dung thật sự |
|---|---|
| `src/` (1553 files) | React 19 + react-router 7 + Zustand + TanStack Query. UI chat/terminal/files/browser/settings/automations/backends. |
| `src/api/` | Service layer gọi Agent Server qua `@openhands/typescript-client` (npm 1.59.0). Chứa `agent-server-adapter.ts` — nơi **lắp ráp request start conversation** (quan trọng nhất). |
| `src/types/agent-server/` | **Mirror TS của event schema** của SDK (Action/Observation/Event). Tài liệu tham chiếu tốt nhất cho event system của ta. |
| `bin/agent-canvas.mjs`, `scripts/dev-*.mjs` | Launcher: spawn `uvx ... agent-server`, `uvx openhands-automation`, static server, ingress proxy. |
| `scripts/ingress.mjs` | Reverse proxy HTTP + WebSocket upgrade: `/api/automation → :18001`, `/api, /sockets → :18000`, `/* → frontend`. |
| `docker/Dockerfile`, `docker/entrypoint.sh` | Image all-in-one `ghcr.io/openhands/agent-canvas` = base `ghcr.io/openhands/agent-server` + automation + frontend. |
| `electron/` + `electron-builder.config.mjs` | Desktop app: Electron bọc cùng stack, bundle thêm `uv` + Node runtime. |
| `helm/agent-canvas/` | StatefulSet chart cho K8s. |
| `tools/canvas_ui_tool.py` | File Python duy nhất: legacy tool `canvas_ui` + đăng ký `FinishTool` cho agent-server (`--import-modules canvas_ui_tool`). |
| `examples/acp-docker/` | docker-compose chạy agent-server trong container + Canvas trỏ vào. |
| `.openhands/setup.sh` | Script setup repo khi chính OpenHands agent làm việc trên repo này (cài uv, npm ci, .env). |
| `.agents/skills/` | Skills (SKILL.md) dành cho contributor-agent: `frontend-api-contracts`, `local-stack-runtime`, `desktop-electron`, `e2e-testing`... |
| `config/defaults.json` | Single source of truth: version pins, ports (18000/18001/8000), paths persistence, image names. |
| `docs/` | `architecture.md`, `ACP_AGENTS.md`, `SELF_HOSTING.md`, `DefenseClaw.md`, `DEVELOPMENT.md`. |

Không có `sdk/` trong repo. `node_modules` chưa cài nên không đọc được source `@openhands/typescript-client`; phân tích contract dựa trên code TS trong `src/` sử dụng client đó.

---

## 1. Kiến trúc tổng thể

### 1.1 Bốn repo, một hệ thống
Theo `README.md` và `AGENTS.md` ("Repository boundaries"):

| Repo | Sở hữu |
|---|---|
| `OpenHands/OpenHands` (repo này) | Agent Canvas UI, frontend state, backend selection, local-stack orchestration |
| `OpenHands/software-agent-sdk` | **Python SDK, Agent Server, agents/tools, conversations, workspaces, events, canonical REST/WebSocket API**, và `clients/typescript/` |
| `OpenHands/typescript-client` | Client TS sinh từ OpenAPI của Agent Server |
| `OpenHands/automation` | Automation (cron/webhook/dispatch) |

Chiều phụ thuộc: *Agent Server contract → TypeScript client → Agent Canvas*. AGENTS.md cấm tái hiện endpoint trong Canvas ("Do not reimplement Agent Server endpoints or contracts in Canvas").

### 1.2 Topology runtime local (`docs/architecture.md`, `docs/SELF_HOSTING.md`)
```
Browser ──> ingress :8000 ──┬── /api/automation/* ──> openhands-automation :18001 (uvx)
                            ├── /api/*, /sockets/* ──> openhands-agent-server :18000 (uvx, Python/FastAPI)
                            ├── /vscode/*          ──> openvscode-server (agent-server spawn, port = agent-server+1)
                            └── /*                 ──> static frontend (sirv) hoặc Vite :3001
```
- `bin/agent-canvas.mjs` → `scripts/dev-with-automation.mjs::main()` → `scripts/dev-safe.mjs::buildAgentServerCommand()` spawn:
  ```
  uvx --python ">=3.12,<3.15" --from openhands-agent-server==1.59.0 \
      --with openhands-sdk==1.59.0 --with openhands-tools==1.59.0 --with openhands-workspace==1.59.0 \
      agent-server --import-modules canvas_ui_tool
  ```
  (`scripts/dev-safe.mjs` dòng ~440-560). Hỗ trợ `OH_AGENT_SERVER_LOCAL_PATH` (editable checkout SDK), `OH_AGENT_SERVER_GIT_REF`, `OH_AGENT_SERVER_VERSION`.
- Env truyền cho agent-server (`buildAgentServerEnv`, dòng ~960-1035): `OH_PERSISTENCE_DIR`, `OH_CONVERSATIONS_PATH`, `OH_BASH_EVENTS_DIR`, `OH_VSCODE_PORT/BASE_PATH`, `OH_SECRET_KEY`, `OH_SESSION_API_KEYS_0`, `AGENT_SERVER_URL`, `OH_EXTRA_PYTHON_PATH=tools/`, `EXTENSIONS_PATH=node_modules/@openhands/extensions`, `PYTHONUTF8=1`, và passthrough `OH_CONVERSATION_RUNTIME*`.

### 1.3 Backend registry — nhiều backend, một UI
`src/api/backend-registry/` (`types.ts`, `active-store.ts`, `auth.ts`, `health-store.ts`): UI lưu danh sách backend `{kind: "local" | "cloud", host, apiKey}` trong localStorage, switch runtime. Mọi service đều rẽ nhánh `getActiveBackend().backend.kind === "cloud"` (gọi `callCloudProxy` với Bearer token) hoặc local (gọi typed client với `X-Session-API-Key`). Đây là mẫu **"Provider adapter theo backend"** rất gần với ý tưởng Provider adapter của AI WORKSPACE.

### 1.4 Tầng frontend
- `src/api/README.md`: services là object tĩnh, bắt buộc bọc trong TanStack Query hooks (`src/hooks/query`, `src/hooks/mutation`), không gọi trực tiếp từ component. Có test `no-direct-agent-server-calls.test.ts` chặn gọi HTTP thẳng.
- State: Zustand stores `src/stores/` (`use-event-store.ts`, `conversation-store.ts`, `agent-store.ts`, `metrics-store.ts`, `secret-request-store.ts`...).
- WebSocket: `src/hooks/use-websocket.ts` (reconnect exponential backoff 1s→30s + jitter 30%, handshake watchdog) và `src/contexts/conversation-websocket-context.tsx` (route frame, streaming delta batcher, seq cursor).
- Build lib: `npm run build:lib` xuất entrypoints `browser/conversation/files/settings/sidebar/terminal/i18n` để nhúng Canvas vào host app (`docs/DEVELOPMENT.md`, "embedding Canvas components").

---

## 2. Agent definition & management

### 2.1 Hai "agent kind" (`src/types/settings.ts`)
```ts
export type AgentKind = "openhands" | "acp";
```
- `"openhands"`: Agent LLM-driven của SDK; áp dụng `agent_settings.llm`, `condenser`, `mcp_config`, `tools`.
- `"acp"`: conversation do **subprocess ACP** (Agent Client Protocol, JSON-RPC over stdio) điều khiển: Claude Code, Codex, Gemini CLI, Pi, OpenCode hoặc Custom. LLM/condenser bị bỏ qua; dùng `acp_command`, `acp_args`, `acp_model`, `acp_server`.

`docs/ACP_AGENTS.md` liệt kê command mặc định (nguồn: `openhands.sdk.settings.acp_providers`, mirror vào ts-client; `src/constants/acp-providers.ts` chỉ thêm icon/i18n):

| Provider | Command |
|---|---|
| Claude Code | `npx -y @agentclientprotocol/claude-agent-acp` |
| Codex | `npx -y @agentclientprotocol/codex-acp` |
| Gemini CLI | `npx -y @google/gemini-cli --acp` |
| Pi | `npx -y --package=pi-acp --package=@earendil-works/pi-coding-agent pi-acp` |
| OpenCode | `npx -y opencode-ai acp` |

**Rất đáng chú ý cho AI WORKSPACE**: OpenHands đã giải chính xác bài toán "Claude Code + Codex CLI làm agent brain" bằng **ACP** thay vì parse stdout CLI. Agent Server spawn CLI, relay từng turn, nhận `ACPToolCallEvent` chuẩn hoá (`tool_kind: read|edit|delete|move|search|execute|think|fetch|switch_mode|other`, `status: pending|in_progress|completed|failed`, `raw_input/raw_output`, `content[]` blocks kiểu `diff`/`terminal`). Xem `src/types/agent-server/core/events/acp-tool-call-event.ts`. Lưu ý trong docs: `npx -y @openai/codex acp` **không** phải ACP server (codex CLI không có subcommand `acp`, treo handshake) — phải dùng wrapper `@agentclientprotocol/codex-acp`.

### 2.2 Agent Profiles (định nghĩa agent có thể tái sử dụng)
`src/api/agent-profiles-service/agent-profiles-service.api.ts` bọc `AgentProfilesClient` → `/api/agent-profiles` (agent-server ≥1.29). Profile `default` được server seed. Profile chỉ chứa refs (LLM profile ref `llm_profile_ref`, `public_skills` allow-list, `disabled_skills` deny-list, `meta_profile_ref` cho Model Router, `enable_classify_and_switch_llm_tool`) — "secret-free at rest". Start conversation gửi `agent_profile_id` + `agent_launch_additions { system_message_suffix_append, llm_profile_ref, skills[] }` (`agent-server-adapter.ts` ~1030-1050).

### 2.3 System prompt
- Không có system prompt trong repo này; nó nằm ở SDK (template Jinja `system_prompt_filename`). Canvas chỉ **append suffix**: `AgentContext.system_message_suffix` nhận khối `<RUNTIME_SERVICES>` (`buildRuntimeServicesSystemSuffix`, `scripts/runtime-services-info.mjs`) cho agent biết URL agent-server/ingress/automation từ góc nhìn sandbox, và `PLANNING_AGENT_INSTRUCTION` cho planner.
- Event `SystemPromptEvent` (`system-event.ts`) ghi lại `system_prompt`, `tools[]` (OpenAI tool format) và `dynamic_context` (datetime, skills, runtime services, secrets) — rất tiện để audit.
- Planner là ví dụ hiếm gửi **raw agent spec** (`buildStartPlanningConversationRequest`, ~1267-1360):
  ```ts
  agent: { kind: "Agent", llm,
           tools: [{name:"glob",params:{}},{name:"grep",params:{}},{name:"planning_file_editor", params:{plan_path}}],
           system_prompt_filename: PLANNING_SYSTEM_PROMPT_FILENAME, system_prompt_kwargs:{plan_structure},
           agent_context,
           condenser:{kind:"LLMSummarizingCondenser", llm:{...llm, usage_id:"planning_condenser"}, max_size:100, keep_first:6} }
  ```
  → cho thấy shape agent spec của SDK: `kind`, `llm`, `tools[{name, params}]`, `system_prompt_filename/kwargs`, `agent_context`, `condenser`.

### 2.4 Skills / microagents
- Thuật ngữ cũ "microagents" đã đổi thành **skills** (`SkillType = "repo" | "knowledge" | "agentskills"`, `SkillInfo { name, triggers[], content, allowed_tools, disable_model_invocation, is_agentskills_format }` — `src/types/settings.ts`).
- Nguồn skills: (a) public catalog bundle từ npm `@openhands/extensions` (build-time snapshot, `src/api/skills-service.ts`), agent-server đọc cùng thư mục qua `EXTENSIONS_PATH`; (b) user `~/.agents/skills/`; (c) project `.agents/skills/` trong workspace (`SkillsClient.getSkills({load_user, load_project, project_dir})`). Định dạng `SKILL.md` frontmatter (tương thích Anthropic agentskills).
- Thư mục `.openhands/` trong repo chỉ còn `setup.sh` (chạy trước khi agent làm việc trên repo) — thay cho `microagents/` cũ. Hooks workspace đọc từ `<project>/.openhands/hooks.json` (`src/api/hooks-service.ts`, server "never walks parents").

---

## 3. Inter-agent / delegation

Ba cơ chế quan sát được:

1. **`TaskAction` / `TaskObservation`** (SDK "task" tool, `core/base/action.ts`): `{ prompt, subagent_type, description?, resume? }` — spawn subagent trong cùng Agent Server (kiểu Claude Code Agent tool). `agent_definitions` được truyền trong start payload (`payload.agent_definitions = conversationSettings.agent_definitions`).
2. **`launch_child_conversation` client tool** (`src/api/launch-child-conversation-client-tool.ts`): agent gọi tool → agent-server ghi `ActionEvent` kind `LaunchChildConversationAction {target:"local"|"cloud", task, title?, repository?, branch?, isolation:"worktree"|"shared"}` → **frontend** intercept qua WebSocket và tự tạo conversation mới với `parent_conversation_id`; kết quả trả lại agent bằng message user prefix `CHILD_CONVERSATION_RESULT_PREFIX`. Child "runs on its own... cannot see this conversation's history — everything it needs must be in the task brief". Server theo dõi `sub_conversation_ids[]` (agent-server ≥1.37.1).
3. **Planning conversation** (plan mode): conversation riêng với raw planner agent, cùng workspace, `parent_conversation_id`, output duy nhất là `PLAN.md` qua `PlanningFileEditorAction`.

Ngoài ra `SwitchLLMAction` / `ClassifyAndSwitchLLM` (Model Router meta-profile) đổi LLM profile giữa chừng, không phải delegation nhưng là "routing".

**Nhận xét**: delegation của OpenHands là *conversation-as-unit* (mỗi agent = 1 conversation, quan hệ cha-con qua id + tags), không có message bus giữa agent. Mô hình "AI software company" của ta cần thêm lớp orchestration (task board, hand-off) phía trên — OpenHands không cung cấp.

---

## 4. Session / conversation state, event stream, persistence, resume

### 4.1 Lifecycle
- Start: `POST /api/conversations` với payload `StartConversationPayloadBase` (`agent-server-adapter.ts` ~1017-1035):
  ```ts
  { workspace, confirmation_policy, security_analyzer?, initial_message?, max_iterations /* default 500 */,
    stuck_detection: true, autotitle, title_llm_profile?, worktree, secrets_encrypted?,
    conversation_id?, parent_conversation_id?, secrets?: Record<string, LookupSecret>, tags?,
    client_tools: ClientToolSpec[], tool_module_qualnames?, plugins?, hook_config?, agent_definitions? }
  ```
- Trạng thái: `ExecutionStatus = idle | running | paused | waiting_for_confirmation | finished | error | stuck` (`core/base/common.ts`). Frontend có thêm `AgentState` cũ (`src/types/agent-state.tsx`), `ConversationStatus`, `RuntimeStatus` (`STATUS$BUILDING_RUNTIME`, `STATUS$SETTING_UP_SKILLS`, `STATUS$ERROR_LLM_AUTHENTICATION`...). Cloud có `AppConversationStartTaskStatus` (`WAITING_FOR_SANDBOX → PREPARING_REPOSITORY → RUNNING_SETUP_SCRIPT → SETTING_UP_GIT_HOOKS → SETTING_UP_SKILLS → STARTING_CONVERSATION → READY`).
- Các thao tác qua `ConversationClient` (`agent-server-conversation-service.api.ts`): `sendMessage`, pause/run, `respondToConfirmation`, `forkConversation`, `condenseConversation` (`POST /api/conversations/{id}/condense`), `deleteConversation`, `PATCH /api/conversations/{id}` (title, **tags thay toàn bộ map — phải read-merge-write**), `getEventCount`, search events, `switchProfile`, `switchAcpModel`, `getVSCodeUrl`.
- Stats: `ConversationStateUpdateEvent{key:"stats"}` → `usage_to_metrics: Record<usage_id, LLMMetrics{accumulated_cost, max_budget_per_task, token_usages[], response_latencies[]}>`, với usage_id như `"default"`, `"condenser"`, `"profile:<name>:<uuid>"` (`conversation-state-event.ts`).

### 4.2 Event stream — session socket protocol (`src/types/agent-server/session-frames.ts`)
`/sockets/session/{conversation_id}?after_seq=N`, auth bằng message đầu tiên sau `onopen` (`sendWebSocketAuth`). Envelope **không phải** Event:
```ts
type SessionFrame =
  | { type:"sync"; from_seq?; through_seq? }            // gửi 1 lần trước replay
  | { type:"durable"; seq:number; event }               // event đã persist, seq = log index
  | { type:"transient"; event }                         // publish nhưng không persist
  | { type:"item_started"; item_id; attempt?; anchor_seq? }
  | { type:"delta"; item_id; attempt?; order; kind?:"text"|"reasoning"; content }
  | { type:"item_aborted"; item_id; attempt?; reason }
  | { type:"error"; code; detail };
```
- **Resume**: client giữ `SeqCursor`, reconnect gửi `after_seq` → server replay backlog durable. Progress frames không replay; slot streaming treo sẽ bị discard (`conversation-websocket-context.tsx` ~1150-1180). `after_seq` thay cho cặp cũ `resend_mode/after_timestamp` (so sánh timestamp naive bị lỗi).
- Streaming: `delta` được batch ≤1 commit/frame (`streaming-delta-batcher`), hiển thị qua `StreamingDeltaEvent` client-minted, retire khi `event.id === item_id`.
- History: `RemoteEventsList.search({limit≤100, page_id, sort_order, timestamp__gte/lt})` (`src/api/event-service/event-service.api.ts`).
- Cloud mode tách 2 host: App API (history persisted) và runtime sandbox (live endpoints, `X-Session-API-Key`, CORS allowlist).

### 4.3 Persistence
- Server-side file store: `~/.openhands/agent-canvas/conversations/<id_hex>/` (events log + metadata; `OH_CONVERSATIONS_PATH`), `bash_events/`, settings/secrets mã hoá bằng `OH_SECRET_KEY`, automation SQLite `automation/automations.db`. Docker `VOLUME ["/home/openhands/.openhands", "/projects"]`.
- Workspace mỗi conversation: `<VITE_WORKING_DIR>/<id_hex>` (mặc định `<state>/workspaces`), tuỳ chọn git **worktree** per conversation (`worktree: true`, `worktree_options{base, ref, branch_prefix:"openhands/"}`; tắt khi runtime docker).
- Metadata client-side bổ sung (`conversation-metadata-store.ts`): `active_profile`, `selected_workspace`; pin/archive giờ lưu bằng server tags (#18697).
- Condensation: `CondensationEvent{forgotten_event_ids[], summary, summary_offset}`, `CondensationSummaryEvent` — context window quản lý bằng condenser trong SDK, UI chỉ render.
- Persisted conversation lưu `tool_module_qualnames` (map tool name → Python module) để server import lại khi restore — lý do `tools/canvas_ui_tool.py` phải tồn tại cho conversation cũ.

---

## 5. Runtime: bash / file edit / browser, Docker, giới hạn tài nguyên

### 5.1 Nơi thực thi
Không có trong repo này — nằm ở `openhands-tools` + `openhands-workspace` (SDK). Canvas chỉ chọn **workspace kind** trong payload:
```ts
workspace: executionRuntime === "docker"
  ? { kind: "DockerExecutionWorkspace", working_dir: DOCKER_EXECUTION_WORKING_DIR }
  : { kind: "LocalWorkspace", working_dir }
```
và đọc `GET /server_info.execution_runtime` để biết server đang chạy local hay docker (`agent-server-compatibility.ts`).

### 5.2 Ba mức sandbox (README "Option 1/2/3")
1. **Không sandbox**: agent-server chạy thẳng trên host (uvx) — "full access to your filesystem".
2. **Một container all-in-one** (`docker/Dockerfile`): base image `ghcr.io/openhands/agent-server:<ver>-python` (đã có tmux, openvscode-server, browser tools), + `uv pip install openhands-automation openhands-tools openhands-agent-server`, + frontend static. `tini` làm init, user `openhands`, mount `~/.openhands` và `/projects`. apt từ Debian snapshot cố định.
3. **Mỗi conversation một container** (`OH_CONVERSATION_RUNTIME=docker`): outer agent-server trên host spawn container con cho từng conversation, mount workspace + persisted state. Env điều khiển (passthrough trong `buildAgentServerEnv`): `OH_CONVERSATION_IMAGE`, `OH_CONVERSATION_IMAGE_HAS_BROWSER`, **`OH_CONVERSATION_CONTAINER_MEMORY`, `OH_CONVERSATION_CONTAINER_CPUS`, `OH_CONVERSATION_CONTAINER_PIDS_LIMIT`, `OH_CONVERSATION_CONTAINER_STARTUP_TIMEOUT`**, `OH_CONVERSATION_RUNTIME_SELECTABLE`. Per-conversation override qua settings `container_memory`, `container_cpus`, `container_browser` → payload `container: {memory, cpus, browser}` (`getConversationRuntimeOptions`). Windows native **không hỗ trợ** mode 3 (cần WSL2, POSIX uid/gid — `README.windows.md`).

### 5.3 Action/Observation model (tool runtime)
Xem mục 10. Điểm thực thi: `ExecuteBashAction{command, is_input, timeout, reset}` (tmux session, hỗ trợ `C-c`, soft timeout trả `exit_code -1` rồi gửi command rỗng để xem thêm log); `FileEditorAction{command: view|create|str_replace|insert|undo_edit, path, old_str, new_str, insert_line, view_range}` (giống Anthropic text_editor); `Glob`, `Grep`; `Browser*` (navigate/click/type/get_state/get_content/scroll/tabs — browser-use style, index-based) trả `BrowserObservation{output, error, screenshot_data}`; `MCPToolAction{data}`; `TaskTrackerAction{command: view|plan, task_list[]}`; `InvokeSkillAction{name}`; `FinishAction{message}`; `ThinkAction{thought}`.

Ngoài agent, UI còn chạy lệnh trực tiếp trong runtime: `RemoteWorkspace.executeCommand(command, cwd, timeout)` (`src/api/runtime-service/agent-server-runtime-service.ts`) và `BashClient.searchEvents` (`bash-service.api.ts`, event `BashCommand`/`BashOutput` phân trang) — tức Agent Server expose **bash-as-a-service** độc lập conversation, hữu ích cho terminal tab và automation.

### 5.4 Editor & browser đi kèm
Agent-server tự spawn **openvscode-server** (`OH_VSCODE_PORT`, `OH_VSCODE_BASE_PATH=/vscode`, `/api/vscode/url`) và ingress route cùng origin. Lưu ý bảo mật trong `SELF_HOSTING.md`: cùng origin ⇒ editor đọc được `localStorage` chứa session key của mọi backend (#16492). Browser tool tắt được bằng `OH_ENABLE_BROWSER=false`.

---

## 6. LLM provider integration

- Hoàn toàn ở SDK (LiteLLM-based: `Message.tool_calls` kiểu `ChatCompletionMessageToolCall`, `ChatCompletionToolParam`, `thinking_blocks` Anthropic, `reasoning_content` — `core/base/event.ts` ghi rõ "Tool call structure from LiteLLM").
- Canvas: settings LLM qua `PATCH /api/settings` với `agent_settings_diff` / `conversation_settings_diff` / `misc_settings_diff` (deep-merge) và schema động từ `/api/settings/agent-schema` + `/api/settings/conversation-schema` (`SettingsSchema{sections[{fields[{key, value_type, choices, secret, prominence, depends_on, required}]}]}` — `src/types/settings.ts`). UI settings được **render từ schema server trả về**, không hard-code — mẫu đáng học.
- **LLM Profiles** (`src/api/profiles-service`): nhiều cấu hình LLM có tên, conversation gắn `llm_profile_ref`; `SwitchLLMAction` đổi giữa chừng; **Meta-profiles / Model Router** (`meta-profiles-service`, `route_task_to_model`, `run_router_at_conversation_start`).
- Secrets LLM: GET settings với header `X-Expose-Secrets: encrypted` trả cipher-text an toàn để round-trip, start conversation gửi `secrets_encrypted: true` để server giải mã (`settings-service.api.ts` ~130-160, ~645-680). Cache settings theo `connectionRevision` để payload mã hoá của backend này không bị dùng cho backend khác.
- `option-service` đọc models/providers từ endpoint LLM của agent-server; `llm-balance-service`, `llm-subscription-service` cho Cloud.
- ACP: model do CLI báo (`POST /api/acp/models`), Canvas "keeps no list of models". Cảnh báo trong docs: không đặt `ANTHROPIC_BASE_URL` cùng `CLAUDE_CODE_OAUTH_TOKEN` (base URL LiteLLM phá bearer auth).

---

## 7. Security model

| Lớp | Cơ chế | File |
|---|---|---|
| Auth API | `X-Session-API-Key` (64-char, persist `~/.openhands/agent-canvas/session-api-key.txt`, env `OH_SESSION_API_KEYS_0`). Local bind loopback + inject key vào HTML (`window.__AGENT_CANVAS_SESSION_API_KEY__`); `--public` thì không inject, user nhập tay. `AGENT_CANVAS_ALLOW_LAN_SESSION_KEY` opt-in. | `bin/agent-canvas.mjs`, `scripts/bind-host.mjs`, `docker/entrypoint.sh` |
| Mã hoá settings/secrets | `OH_SECRET_KEY` auto-gen 32 bytes, persist `secret-key.txt` chmod 600; thiếu key ⇒ server trả 503 khi tạo conversation | `entrypoint.sh`, `dev-safe.mjs` |
| Confirmation mode | Payload `confirmation_policy`: `{kind:"NeverConfirm"}` / `{kind:"AlwaysConfirm"}` / `{kind:"ConfirmRisky", threshold:"HIGH", confirm_unknown:true}`; `security_analyzer`: `LLMSecurityAnalyzer` / `PatternSecurityAnalyzer` / `PolicyRailSecurityAnalyzer`. Mỗi `ActionEvent` có `security_risk: UNKNOWN|LOW|MEDIUM|HIGH` (LLM dự đoán, nằm trong `tool_call` args). Agent dừng ở `waiting_for_confirmation`; UI gọi `respondToConfirmation({accept, reason})` → `UserRejectObservation{rejection_reason}`. | `agent-server-adapter.ts` ~768-795, `event-service.api.ts` |
| Hooks | `PreToolUse/PostToolUse/UserPromptSubmit/SessionStart/SessionEnd/Stop` từ `.openhands/hooks.json`; `HookExecutionEvent{blocked, reason, exit_code, stdout}` — hook có thể **block action** (giống Claude Code hooks). | `hook-execution-event.ts`, `hooks-service.ts` |
| Secrets runtime | `secrets: Record<name, {kind:"LookupSecret", url:"/api/settings/secrets/<name>", headers}>` — server **tự fetch giá trị lúc spawn** (off event loop, SDK#3510), không bao giờ đi qua browser/agent; expose thành env var tên = tên secret; SDK `sanitized_env()` strip khỏi bash subprocess; ACP `acp_file_secrets` materialize `CODEX_AUTH_JSON` → `~/.codex/auth.json`, `PI_AUTH_JSON`, `GOOGLE_APPLICATION_CREDENTIALS_JSON`… | `buildCustomSecrets`, `docs/ACP_AGENTS.md` |
| Agent xin secret | client tool `request_secret{name, reason}` → UI form, giá trị không qua agent, kết quả báo lại bằng message prefix | `request-secret-client-tool.ts` |
| MCP credential | redact trong error/tool result (`redactMcpSecrets`), OAuth flow `/api/mcp/oauth/*` với popup + polling job | `mcp-service.api.ts` |
| Electron | `contextIsolation: true`, `nodeIntegration: false`, preload chỉ expose boot-log IPC; `setWindowOpenHandler` + `will-navigate` policy (`electron/lib/window-url-policy.mjs`); `safeStorage` cho MARS token | `electron/main.mjs`, `preload.cjs` |
| Supply chain | npm `--min-release-age=7`, exact pin, Debian snapshot apt, `tini`, Dependabot cooldown | `AGENTS.md`, `Dockerfile` |
| Governance ngoài | DefenseClaw: LLM guardrail proxy qua `base_url`, scan skills/MCP, audit export | `docs/DefenseClaw.md` |

Điểm yếu tự nhận: mode không sandbox = full host access; editor same-origin đọc localStorage; `/sockets` auth bằng message đầu tiên; ACP conversations cùng provider trong 1 container chia sẻ HOME (race auth/lock files, `acp_isolate_data_dir` chưa expose).

---

## 8. Skills / MCP / plugins / hooks / client tools

- **MCP**: `MCPServerConfig{type: sse|stdio|shttp, url|command+args+env, headers, auth{strategy: oauth2|...}, enabled}` lưu trong `agent_settings.mcp_config`; test server `POST /api/mcp/test` (trả `tools[]`, `tool_result`, `error_kind: connection|timeout|credentials|authorization...`), OAuth popup flow. Agent nhận `MCPToolAction{data}` / `MCPToolObservation{content[], is_error, tool_name}`. Marketplace MCP (Slack/GitHub/Figma stdio qua `npx`). `mcp-health` probe định kỳ.
- **Plugins**: `PluginSpec{source:"github:owner/repo"|git|path, ref, repo_path, parameters}` gửi trong `plugins[]` của start request; agent-server có marketplace (`PluginsClient.getPluginsMarketplace`) và ambient plugins `~/.agents/plugins`, `~/.openhands/plugins` (`src/api/plugins-service.ts`). Plugin bundle skills + files + hooks.
- **Client tools** (`client_tools: ClientToolSpec[]` — `{name, description, parameters(JSON Schema), annotations{readOnlyHint,destructiveHint,idempotentHint,openWorldHint}}`): tool do **frontend** thực thi; server chỉ validate schema (tên + kiểu, không enum), ghi ActionEvent, trả `ClientToolObservation` ack; frontend nghe WebSocket và dispatch (`src/services/canvas-ui.ts`). Server cache schema theo tên trong process (`ClientToolSchemaConflictError`). Bốn tool: `canvas_ui_control`, `launch_child_conversation`, `request_secret`, `automation_form_update`. Đây là mẫu **"UI-side tool"** rất hay để agent điều khiển UI của AI WORKSPACE (mở tab, highlight diff, xin secret).
- **Skills**: xem 2.4; profile chọn `public_skills`; skill được invoke bằng `/skill` trong message → `findInvokedCatalogSkill` thêm `agent_launch_additions.skills`.
- **Automations** (repo `OpenHands/automation`, không trong repo này): cron/webhook, dispatch conversation về Agent Server, KV state `AUTOMATION_KV_SECRET`, tích hợp Slack/GitHub/Linear. Canvas chỉ có UI + `automation-form-session`.
- **Canvas extensions** (`src/extensions/`, `src/types/canvas-extension.ts`): module UI nạp động.

---

## 9. Electron desktop packaging (đối chiếu với quyết định Tauri)

Cách OpenHands làm (`electron/main.mjs`, `electron-builder.config.mjs`, `scripts/download-uv.mjs`, `download-node.mjs`):
- Electron **chỉ là shell**: main process gọi lại chính `scripts/dev-with-automation.mjs::main()` để spawn uvx agent-server + automation + static server + ingress, show `loading.html` (stream boot log qua IPC) rồi `loadURL("http://localhost:8000")`. Không có IPC nghiệp vụ.
- **Bundle runtime bên ngoài**: tải `uv/uvx` và **full Node.js distribution** vào `extraResources` (`bin-<arch>/`, `node-<arch>/`), prepend vào `PATH` lúc chạy (`injectBundledUv`, `injectBundledNode`). Lý do ghi rõ: app mở từ Finder có PATH tối thiểu `/usr/bin:/bin`; `ELECTRON_RUN_AS_NODE` **không** đủ tin cậy cho stdio JSON-RPC MCP servers ("McpError: Connection closed") → phải ship Node thật, kèm `node_modules/npm` để `npx` chạy.
- `asar: false` (scripts spawn như child process), `afterPack` xoá `node_modules` ~600MB rồi copy lại closure của `["sirv","httpxy","ws","@openhands/extensions"]` (~6-16MB).
- Targets: macOS dmg (universal, @electron/universal merge x64+arm64), Windows NSIS x64 (`appId dev.openhands.agent-canvas`, `setAppUserModelId`), Linux AppImage + deb.
- Shutdown: `before-quit` → `signalProcessTree` SIGTERM toàn bộ process tree (`scripts/dev-process-utils.mjs`), hai pha preventDefault/quit.
- Lần đầu chạy uvx tải Python + packages ⇒ chờ tới 10 phút (`waitForAgentServer` poll `/server_info` tới khi 200/401, timeout 600s; 502 từ ingress nghĩa là upstream chưa lên).

**Hàm ý cho Tauri của ta**:
1. Mô hình "shell + sidecar backend HTTP local + webview load localhost" tương thích hoàn toàn với Tauri (sidecar binary + `tauri-plugin-shell`), nhẹ hơn nhiều so với Electron ~150MB. Tauri không có afterPack "node_modules" problem.
2. Bài học lớn nhất: **phải ship runtime cho agent brain** (Node cho Claude Code/Codex CLI qua npx, hoặc binary CLI) và quản lý PATH; đừng dựa vào PATH của OS khi app được mở từ GUI. Trên Windows cần `npm.cmd/npx.cmd` + `node_modules/npm`.
3. Cần splash + boot log stream (first run chậm), process-tree kill lúc quit, và đọc `/server_info` để biết backend sẵn sàng (không chỉ port mở).
4. Windows: OpenHands chạy native (không Docker) được nhờ `PYTHONUTF8=1`, nhưng per-conversation Docker cần WSL2. Với NestJS backend + Docker Desktop của ta, nên thiết kế fallback "no-sandbox local" như Option 1 và cảnh báo rõ.

---

## 10. Event schema (tham chiếu cho event system của AI WORKSPACE)

Nguồn: `src/types/agent-server/core/**` (mirror của `openhands.sdk.event`). Thiết kế nổi bật:

```ts
interface BaseEvent { id: EventID /*ULID*/; timestamp: string; source: "agent"|"user"|"environment"|"hook" }

interface ActionEvent<T extends Action> extends BaseEvent {
  thought: TextContent[]; reasoning_content?; thinking_blocks: (ThinkingBlock|RedactedThinkingBlock)[];
  action: T;                       // discriminated by action.kind, vd "ExecuteBashAction"
  tool_name: string; tool_call_id: ToolCallID;
  tool_call: ChatCompletionMessageToolCall;   // bản gốc từ LLM (có thể chứa security_risk)
  llm_response_id: EventID;        // nhóm các tool call song song cùng 1 response
  security_risk: "UNKNOWN"|"LOW"|"MEDIUM"|"HIGH";
  critic_result?: CriticResult; summary?: string;
}
interface ObservationEvent<T extends Observation> extends BaseEvent {
  source: "environment"; tool_name; tool_call_id; action_id: EventID; observation: T  // observation.kind = "ExecuteBashObservation"...
}
interface UserRejectObservation { rejection_reason; action_id }
interface MessageEvent { llm_message: Message; activated_skills: string[]; extended_content: TextContent[]; critic_result? }
interface SystemPromptEvent { system_prompt: TextContent; tools: ChatCompletionToolParam[]; dynamic_context? }
interface ConversationStateUpdateEvent { key: "full_state"|"execution_status"|"stats"|"goal"; value }
interface CondensationEvent { forgotten_event_ids[]; summary?; summary_offset? }
interface HookExecutionEvent { hook_event_type; hook_command; success; blocked; exit_code; reason; tool_name; action_id; stdout; stderr; hook_input }
interface ACPToolCallEvent { tool_call_id; title; tool_kind; status; raw_input; raw_output; content[]; is_error }
interface PauseEvent { source:"user" }  interface ServerErrorEvent { code; detail }  // + AgentErrorEvent, ConversationErrorEvent từ ts-client
interface StreamingDeltaEvent { content; reasoning_content; attempt }   // client-only, không persist
```
Danh sách `kind` (`core/base/base.ts`): Action = `{MCPTool,Finish,Think,ExecuteBash,Terminal,FileEditor,StrReplaceEditor,TaskTracker,PlanningFileEditor,InvokeSkill,SwitchLLM,ClassifyAndSwitchLLM}Action`, `Browser{Navigate,Click,Type,GetState,GetContent,Scroll,GoBack,ListTabs,SwitchTab,CloseTab}Action`, `GlobAction`, `GrepAction`, `TaskAction`, + client tool kinds. Observation = tương ứng + `BrowserObservation`, `TerminalObservation`, `TaskObservation`, `ClientToolObservation`.

`ExecuteBashObservation` kèm `CmdOutputMetadata{exit_code, pid, username, hostname, working_dir, py_interpreter_path, prefix, suffix}`; `TaskItem{title, notes, status: todo|in_progress|done}`; `Message.content` là mảng `TextContent{text, cache_prompt?}` | `ImageContent{image_urls[]}`; `TokenUsage{prompt_tokens, completion_tokens, cache_read_tokens, cache_write_tokens, reasoning_tokens, context_window, per_turn_token, response_id}`.

Nguyên tắc rút ra cho ta: (1) tách **Event envelope** (id/seq/timestamp/source) khỏi **payload** (action/observation có `kind` discriminator); (2) luôn giữ `tool_call_id` + `action_id` để ghép cặp action↔observation; (3) `llm_response_id` cho parallel tool calls; (4) `security_risk` + `critic_result` là metadata trên event, không phải event riêng; (5) tách **durable** vs **transient** vs **delta** ở tầng transport (session frames), persistence chỉ lưu durable với `seq` tăng dần → resume/replay rẻ; (6) state snapshot (`full_state`) + partial update (`execution_status`, `stats`) cùng một event kind; (7) unknown frame/event kind phải được bỏ qua ("the envelope is allowed to grow").

---

## 11. Đánh giá tái sử dụng cho AI WORKSPACE

### 11.1 Dùng trực tiếp (không sửa hoặc sửa rất ít)
- **Agent Server + Docker images** `ghcr.io/openhands/agent-server:<ver>-python` (từ SDK) hoặc all-in-one `ghcr.io/openhands/agent-canvas`: chạy như một **service ngoài**, NestJS gọi qua REST/WS. Có sẵn tmux bash, file editor, browser, VSCode, skills, MCP, hooks, confirmation, per-conversation Docker với CPU/mem/pids limit.
- **`@openhands/typescript-client`** (npm 1.59.0): client typed cho toàn bộ API (ConversationClient, FileClient, BashClient, SettingsClient, SkillsClient, PluginsClient, MCPClient, HooksClient, AgentProfilesClient, RemoteWorkspace, RemoteEventsList, device-flow). Dùng được trong NestJS (browser-compatible, `ws`).
- **Event schema TS** `src/types/agent-server/**` và `session-frames.ts`: copy làm `openhands` namespace trong event model của ta hoặc làm tài liệu thiết kế.
- **`scripts/ingress.mjs`**, `dev-process-utils.mjs`, `logger.mjs`: proxy + process-tree utilities nhỏ, MIT.
- **Config pattern** `config/defaults.json` → generate `defaults.env` cho Docker; `helm/agent-canvas` chart (StatefulSet, probes `/alive`, non-root uid 10001).

### 11.2 Tham khảo (reference-only)
- `src/api/agent-server-adapter.ts`: cách lắp payload (confirmation policy, LookupSecret, client tools, runtime options, planner raw agent).
- `conversation-websocket-context.tsx` + `use-websocket.ts`: reconnect/resume/streaming batching.
- Client tools pattern (`canvas-ui-client-tool.ts`, `launch-child-conversation-client-tool.ts`, `request-secret-client-tool.ts`).
- ACP onboarding/credential flow (`docs/ACP_AGENTS.md`, `src/constants/acp-providers.ts`).
- Electron packaging decisions (mục 9) → map sang Tauri.
- Settings-schema-driven UI, secrets `X-Expose-Secrets: encrypted`.
- `.agents/skills/*/SKILL.md` và `AGENTS.md`: mẫu tài liệu cho contributor-agent rất tốt (ownership table, change checklist, verification-skill independence).

### 11.3 Phải viết mới
- Toàn bộ UI (React Canvas gắn chặt HeroUI/i18n/PostHog/Cloud; `build:lib` cho phép nhúng vài component nhưng chi phí tích hợp cao; nên tự viết UI theo design system của ta).
- Lớp orchestration "AI software company": roles, task board, hand-off nhiều agent, review loop — OpenHands chỉ có parent/child conversation.
- NestJS Provider adapter `OpenHandsProvider` (mục 11.4), mapping event OpenHands → event chuẩn của ta, PostgreSQL persistence (OpenHands lưu file JSON; cần ingest durable frames vào DB của ta).
- Automation (nếu cần) — repo `OpenHands/automation` riêng, SQLite, Python.

### 11.4 Đề xuất tích hợp OpenHands như Provider thứ ba
```
AI WORKSPACE (NestJS)
  ProviderRegistry
    ├── ClaudeCodeProvider  (CLI/SDK)
    ├── CodexProvider       (CLI)
    └── OpenHandsProvider   ──HTTP/WS (X-Session-API-Key)──> openhands-agent-server (uvx hoặc docker, :18000)
          • startSession(spec)  → POST /api/conversations { agent | agent_profile_id,
                                   workspace{kind:"DockerExecutionWorkspace"|"LocalWorkspace"},
                                   confirmation_policy, security_analyzer,
                                   secrets{ name: {kind:"LookupSecret", url: <NestJS secret endpoint>} },
                                   client_tools[...], max_iterations, tags{workspace_id, role, task_id} }
          • send(text)          → ConversationClient.sendMessage
          • stream()            → ws /sockets/session/{id}?after_seq=N → map SessionFrame → WorkspaceEvent
                                   (durable→persist Postgres với seq, delta→SSE/WS tới UI)
          • confirm(accept)     → respondToConfirmation ; pause/run ; fork ; condense
          • exec(cmd)           → RemoteWorkspace.executeCommand (bash-as-a-service cho terminal tab)
          • files               → FileClient (download/upload) ; VSCode url
          • ACP sub-mode        → agent_kind "acp" + acp_command: dùng OpenHands làm *host* cho Claude Code/Codex
                                   trong Docker sandbox (tận dụng credential materialization), thay vì tự quản lý subprocess
```
Rủi ro: wire contract thay đổi nhanh (Canvas pin `minimumAgentServer` 1.57.0, nhiều field mới mỗi release); cần version-gate như `agent-server-compatibility.ts` (`/server_info` version + feature probe, fallback 404). Python + uv là dependency thêm trong deployment của ta. LookupSecret URL phải reachable từ agent-server (hoặc từ trong container con).

---

## 12. Bảng tổng hợp

| Module | Path | Tái sử dụng trực tiếp | Cần chỉnh sửa | Viết mới | Ghi chú |
|---|---|---|---|---|---|
| Agent Server (SDK, Python) | repo `software-agent-sdk`, pin `config/defaults.json` → `uvx openhands-agent-server==1.59.0` | Có — chạy như service ngoài | version-gate contract | – | Nguồn thật của agents/tools/runtime; không vendored |
| Docker image agent-server | `ghcr.io/openhands/agent-server:<ver>-python` (base của `docker/Dockerfile`) | Có | thêm tool/ngôn ngữ nếu cần | – | Có tmux, openvscode, browser, per-conversation docker |
| All-in-one image + entrypoint | `docker/Dockerfile`, `docker/entrypoint.sh` | Một phần (chạy thử nhanh) | bỏ frontend/automation, giữ env/secret-key logic | – | Env `OH_*` là tài liệu cấu hình tốt nhất |
| TypeScript client | npm `@openhands/typescript-client@1.59.0` | Có — dùng trong NestJS | – | – | Browser-compatible, có WS/events/workspace |
| Event schema TS | `src/types/agent-server/core/**`, `openhands-event.ts` | Có — copy làm reference types | map sang event chuẩn của ta | – | Mục 10 |
| Session frame protocol | `src/types/agent-server/session-frames.ts` | Có | – | – | durable/transient/delta + `after_seq` resume |
| WebSocket client | `src/hooks/use-websocket.ts`, `src/contexts/conversation-websocket-context.tsx` | – | port logic reconnect/batcher sang NestJS/React của ta | Có | Reference-only, gắn Zustand |
| Start-conversation builder | `src/api/agent-server-adapter.ts` | – | trích `buildStartConversationRequest`, `getConversationConfirmationPolicy`, `buildCustomSecrets` | Có, trong `OpenHandsProvider` | Mục 4.1, 7 |
| Client tools (UI-side tools) | `src/api/canvas-ui-client-tool.ts`, `launch-child-conversation-client-tool.ts`, `request-secret-client-tool.ts` | Một phần (spec JSON) | đổi tên/description theo UI của ta | dispatcher mới | Pattern rất đáng áp dụng |
| ACP provider registry | `src/constants/acp-providers.ts`, `docs/ACP_AGENTS.md` | Một phần (command list) | – | – | Cách chạy Claude Code/Codex qua ACP trong sandbox |
| Skills / extensions | npm `@openhands/extensions`, `src/api/skills-service.ts`, `.agents/skills/` | Có (catalog SKILL.md) | – | – | Tương thích agentskills format |
| MCP service | `src/api/mcp-service/`, `src/types/mcp-server.ts` | – | dùng `MCPClient` + redact logic | UI mới | OAuth flow qua agent-server |
| Plugins / Hooks | `src/api/plugins-service.ts`, `hooks-service.ts`, `hook-execution-event.ts` | Một phần (contract) | – | UI mới | hooks.json giống Claude Code |
| Settings schema-driven | `src/types/settings.ts`, `settings-service.api.ts` | – | học pattern `agent-schema` + `X-Expose-Secrets` | Có | |
| Backend registry (multi-backend) | `src/api/backend-registry/` | – | ý tưởng cho ProviderRegistry | Có | local/cloud switch |
| Launcher / process mgmt | `bin/agent-canvas.mjs`, `scripts/dev-safe.mjs`, `dev-process-utils.mjs`, `ingress.mjs` | Có (ingress + process utils) | `buildAgentServerCommand/Env` port sang NestJS bootstrap | – | Spawn uvx, PATH, UTF-8 Windows |
| Electron desktop | `electron/`, `electron-builder.config.mjs`, `scripts/download-uv.mjs`, `download-node.mjs` | – | lấy chiến lược bundle uv/Node, splash, process-tree kill | Có (Tauri sidecar) | Mục 9 |
| Helm chart | `helm/agent-canvas/` | Một phần | đổi image/env | – | StatefulSet + probes `/alive` |
| Frontend React (Canvas) | `src/components/**`, `src/stores/**`, `src/routes/**` | Không | `build:lib` nhúng terminal/files nếu muốn | Có | Gắn chặt HeroUI/i18n/PostHog/Cloud |
| Automation | repo `OpenHands/automation` (chỉ pin version ở đây) | Tuỳ chọn | – | Có nếu cần cron/webhook riêng | Python + SQLite |
| Security docs | `docs/SELF_HOSTING.md`, `docs/DefenseClaw.md` | Có (checklist) | – | – | Firewall, API key, same-origin risk |
| Legacy Python tool | `tools/canvas_ui_tool.py` | – | ví dụ viết custom ToolDefinition cho SDK (`Action/Observation/ToolExecutor/register_tool`) | – | Mẫu duy nhất của SDK tool API trong repo |
