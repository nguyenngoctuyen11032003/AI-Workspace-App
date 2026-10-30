# Phân tích ChatDev (1.0 "Virtual Software Company" và 2.0 "DevAll Workflow Builder")

> Agent 01 – ChatDev Research Analyst | Dự án AI WORKSPACE
> Repo tham chiếu: `D:\Code\Tuyen\Agent management\ChatDev` (clone https://github.com/OpenBMB/ChatDev, commit `a2ea4a2`, Apache-2.0)
> Ngày: 2026-10-30

---

## 0. Tóm tắt điều hành (Executive summary)

- Repo hiện tại trên `main` là **ChatDev 2.0 (tên nội bộ "DevAll")**: một **workflow orchestration runtime** bằng Python 3.12 + FastAPI, trong đó mỗi agent là một **node** trong đồ thị có hướng (DAG hoặc có chu trình) được mô tả bằng **YAML**; frontend là **Vue 3 + Vue Flow** để kéo-thả đồ thị và chạy workflow qua WebSocket.
- **ChatDev 1.0 (Virtual Software Company)** vẫn tồn tại trong git: tag `v1.0.0 … v1.1.6` và branch `origin/chatdev1.0`. Lõi 1.0 là **ChatChain → Phase (SimplePhase/ComposedPhase) → RolePlaying (2 agent đối thoại)** cấu hình bằng 3 file JSON: `ChatChainConfig.json`, `PhaseConfig.json`, `RoleConfig.json`.
- Phiên bản 2.0 đã "port" lại 1.0 thành một workflow YAML: `yaml_instance/ChatDev_v1.yaml` (1021 dòng), dùng `agent`/`literal`/`loop_counter`/`passthrough` node + `keyword` edge condition để tái hiện Coding → CodeComplete → CodeReview → Test → Manual.
- Giá trị tái sử dụng cho AI WORKSPACE (TypeScript/NestJS + React + Phaser):
  - **Trực tiếp**: các role prompt & phase prompt 1.0 (`RoleConfig.json`, `PhaseConfig.json`), YAML schema của 2.0 (ý tưởng node/edge/condition/dynamic), protocol WebSocket (`connection`, `log`, `human_input_required`, `artifact_created`, `workflow_completed`, `session_resumed`…), thiết kế `SKILL.md` (frontmatter `name/description/allowed-tools`), bộ sprite 12 nhân vật pixel-art (`frontend/public/sprites/{1..12}-{D,L,R,U}-{1..3}.png`) nếu tuân thủ license.
  - **Tham chiếu kiến trúc**: GraphExecutor + chiến lược DAG/Cycle/MajorityVote, Tarjan SCC + super-node cho vòng lặp, edge-level Map/Tree fan-out, `ResourceManager` semaphore theo loại node, `WorkspaceArtifactHook` quét file mới sau mỗi node, `HumanPromptService`/`PromptChannel`, `TokenTracker`, Registry pattern (node/provider/memory/thinking/condition/processor).
  - **Phải viết lại**: toàn bộ Python runtime (không thể chạy trong NestJS), provider layer (OpenAI/Gemini → cần Claude Code/Codex CLI adapter), session store (in-memory → PostgreSQL), code execution (subprocess không sandbox → Docker sandbox), frontend (Vue → React + Phaser).

---

## 1. Lịch sử git, tag, branch

```
$ git branch -a
* main
  remotes/origin/chatdev1.0      # nhánh giữ ChatDev 1.0
  remotes/origin/macnet          # MacNet (DAG nhiều agent, paper 2406.07155)
  remotes/origin/puppeteer       # Puppeteer (multi-agent orchestration)
$ git tag
v1.0.0 v1.0.1 v1.1.0 … v1.1.6    # 1.0 line (467 commits tới v1.1.6)
v2.0.0 v2.1.0 v2.2.0             # 2.0 line (209 commits trên main)
$ git log --oneline | tail -1
f0db945 initial commit of chatdev 2.0   # main được reset từ "initial commit of chatdev 2.0"
```

Cây file 1.0 (tại `v1.1.6`):

```
CompanyConfig/{Default,Art,Human,Incremental}/ChatChainConfig.json
CompanyConfig/Default/{PhaseConfig.json,RoleConfig.json}
camel/   (fork của CAMEL: agents/, messages/, prompts/, model_backend.py, typing.py)
chatdev/ (chat_chain.py, phase.py, composed_phase.py, chat_env.py, codes.py, roster.py, documents.py, statistics.py)
visualizer/app.py (Flask log viewer + replay)
run.py
```

---

## 2. Kiến trúc tổng thể ChatDev 2.0

### 2.1 Stack

| Lớp | Công nghệ | Bằng chứng |
|---|---|---|
| Backend | Python >=3.12,<3.13, FastAPI 0.124, uvicorn (ws=wsproto), pydantic 2.12, openai SDK, google-genai, fastmcp + mcp, tenacity, faiss-cpu, mem0ai, networkx | `pyproject.toml` |
| Package manager | `uv` (uv.lock), hatchling; `make dev` = `uv run python server_main.py --port 6400` + `npm run dev` | `Makefile`, `Dockerfile` |
| Frontend | Vue 3.5, Vite 7, `@vue-flow/core` 1.47 (+ background/controls/minimap), vue-router 4, vue-i18n 11, js-yaml, markdown-it | `frontend/package.json` |
| Persistence | **Không có DB**. YAML trong `yaml_instance/`, output trong `WareHouse/session_<id>/`, session chỉ in-memory | `server/settings.py`, `session_store.py` |
| Deploy | `compose.yml` 2 service (backend :6400, frontend :5173), bind-mount source | `compose.yml` |
| Tests | pytest, 8 file (security attachment/session/code exec, mem0, websocket) | `tests/` |

### 2.2 Cấu trúc thư mục backend

```
server_main.py        CLI uvicorn (--reload chỉ watch source dirs, exclude WareHouse/)
run.py                CLI chạy 1 workflow không cần server (input() cho human node)
server/
  app.py, bootstrap.py (CORS *, routers, state.init_state())
  state.py            singleton WebSocketManager
  models.py           WorkflowRequest / WorkflowRunRequest (pydantic)
  routes/             health, workflows (CRUD yaml), uploads, artifacts, sessions, batch,
                      execute (async qua WS), execute_sync (/api/workflow/run, SSE), tools, vuegraphs, websocket
  services/           websocket_manager, session_store, session_execution, workflow_run_service,
                      websocket_executor, websocket_logger, prompt_channel, attachment_service,
                      artifact_events/dispatcher, batch_run_service, workflow_storage, vuegraphs_storage
entity/               dataclass config + FIELD_SPECS (schema tự mô tả cho UI form)
  configs/graph.py, configs/node/{agent,human,literal,passthrough,python_runner,subgraph,loop_counter,loop_timer,memory,thinking,tooling,skills}.py
  configs/edge/{edge,edge_condition,edge_processor,dynamic_edge_config}.py
  messages.py         Message / MessageBlock / AttachmentRef / ToolCallPayload
  tool_spec.py        ToolSpec provider-agnostic (to_openai_dict / to_gemini_function)
schema_registry/      registry các schema node/edge/memory/thinking/provider -> export JSON cho frontend
runtime/
  node/registry.py, node/builtin_nodes.py       đăng ký 8 node type
  node/executor/*.py                             AgentNodeExecutor (1361 dòng), Human, Python, Subgraph, Literal, Passthrough, LoopCounter, LoopTimer
  node/agent/providers/{base,openai_provider,gemini_provider}.py
  node/agent/tool/tool_manager.py                function + mcp_remote + mcp_local
  node/agent/skills/manager.py                   Agent Skills (SKILL.md)
  node/agent/memory/{simple,file,blackboard,mem0}_memory.py
  node/agent/thinking/self_reflection.py
  edge/conditions/{keyword,function}, edge/processors/{regex,function}
  sdk.py              run_workflow() cho Python SDK
workflow/
  graph.py            GraphExecutor (862 dòng) – vòng đời chạy đồ thị
  graph_manager.py, topology_builder.py, cycle_manager.py (Tarjan)
  executor/{dag,cycle,parallel,dynamic_edge,resource_manager}.py
  runtime/{runtime_builder,runtime_context,execution_strategy,result_archiver}.py
  hooks/workspace_artifact.py
functions/function_calling/*.py   tool thuần Python được nạp động (file, web, code_executor, uv_related, deep_research…)
functions/edge/conditions.py, functions/edge_processor/transformers.py
utils/                human_prompt, code_execution (MAC_ALLOW_UNSAFE_CODE_EXECUTION), token_tracker, logger, workspace_paths…
yaml_template/design.yaml          schema "self-documented" của toàn bộ YAML
yaml_instance/*.yaml               ~45 workflow mẫu (ChatDev_v1, MACNet_v1, deep_research_v1, demo_*)
.agents/skills/<name>/SKILL.md     3 skill mẫu
mcp_example/mcp_server.py          FastMCP server mẫu
docs/user_guide/{en,zh}/           tài liệu người dùng
```

### 2.3 Luồng chạy một workflow (Web)

```
Browser ──WS /ws──> WebSocketManager.connect() → {"type":"connection","data":{session_id}}
Browser ──POST /api/workflow/execute {yaml_file, task_prompt, session_id, attachments}
  → WorkflowRunService.start_workflow()
     → load_config(yaml) (check_yaml + check_workflow_structure + DesignConfig.from_dict)
     → GraphConfig.from_definition(...) ; GraphContext(config)
     → WebSocketGraphExecutor(graph_context, session_id, ...)        # subclass GraphExecutor
     → await executor.execute_graph_async(task_input)                 # run_in_executor (thread)
        → GraphExecutor.run(): GraphManager.build_graph() → strategy (DAG/Cycle/MajorityVote)
           → _execute_node() cho từng node → NodeExecutor.execute(node, inputs)
           → mỗi output đi qua _process_edge_output() → condition_manager.process() → target.append_input()
     → {"type":"workflow_completed","data":{results, summary, token_usage}}
Mọi log_manager.record_* → WebSocketLogger.add_log → {"type":"log","data":LogEntry}
```

`server/services/websocket_executor.py`:
```python
class WebSocketGraphExecutor(GraphExecutor):
    def _create_logger(self) -> WorkflowLogger:
        return WebSocketLogger(self.websocket_manager, self.session_id, self.graph.name, self.graph.log_level)
    async def execute_graph_async(self, task_prompt):
        await asyncio.get_event_loop().run_in_executor(None, self._execute, task_prompt)
```

### 2.4 Frontend (Vue 3 + Vue Flow)

- Routes: `/` Home, `/tutorial`, `/launch` (chạy workflow, chat log), `/batch-run`, `/workflows/:name?` (Workbench = WorkflowList + WorkflowView).
- `WorkflowView.vue` (2423 dòng): dùng `<VueFlow>` với custom node `WorkflowNode.vue`, custom edge `WorkflowEdge.vue`, `StartNode.vue`; form cấu hình node sinh tự động từ **schema API** (`FormGenerator.vue` + `DynamicFormField.vue` ~4.4k dòng) dựa trên `FIELD_SPECS` của backend (`server/config_schema_router.py`, `utils/schema_exporter.py`). Vị trí node lưu riêng qua `/api/vuegraphs/*` (`vuegraphs_storage.py`), YAML lưu qua `/api/workflows/*`.
- `LaunchView.vue` (3705 dòng): tự mở WS, xử lý `connection`, `session_resumed`, `human_input_required`, `artifact_created`, `log` (INFO → dialogue, WARNING/ERROR → notification), `workflow_completed`, `workflow_cancelled`, `error`.
- Node hiển thị bằng **sprite pixel-art** gán ngẫu nhiên theo node_id (`utils/spriteFetcher.js`): `/sprites/${character}-${stance}-${frame}.png`, 12 nhân vật × 4 hướng (D/L/R/U) × 3 frame, có "walking animation" khi node active. Ý tưởng này gần với virtual office Phaser của AI WORKSPACE.

---

## 3. Agent được định nghĩa và quản lý như thế nào (2.0)

### 3.1 Agent = node type `agent`

Đăng ký trong `runtime/node/builtin_nodes.py`:
```python
register_node_type("agent", config_cls=AgentConfig, executor_cls=AgentNodeExecutor,
    capabilities=NodeCapabilities(default_role_field="role", exposes_tools=True),
    summary="Agent execution node backed by configured LLM/tool providers ...")
register_node_type("human", ..., capabilities=NodeCapabilities(resource_key="node_type:human", resource_limit=1))
register_node_type("python", ..., capabilities=NodeCapabilities(resource_key="node_type:python", resource_limit=1))
```

`entity/configs/node/agent.py`:
```python
@dataclass
class AgentConfig(BaseConfig):
    provider: str                 # "openai" | "gemini" (ProviderRegistry)
    base_url: str                 # mặc định "${BASE_URL}"
    name: str                     # model name, vd gpt-4o
    role: str | None = None       # system prompt
    api_key: str | None = None    # mặc định "${API_KEY}"
    params: Dict[str, Any]        # temperature, max_tokens, protocol: chat|responses ...
    retry: AgentRetryConfig | None     # tenacity: max_attempts=5, wait 1..6s, retry_on_status_codes...
    input_mode: AgentInputMode = MESSAGES   # "prompt" (flatten text) | "messages" (giữ role)
    tooling: List[ToolingConfig]  # function | mcp_remote | mcp_local (+ prefix)
    thinking: ThinkingConfig | None     # reflection
    memories: List[MemoryAttachmentConfig]  # tham chiếu memory store cấp graph
    skills: AgentSkillsConfig | None    # enabled + allow[]
```

Node chung (`entity/configs/node/node.py`): `id`, `type`, `description`, `context_window` (0 = xoá context sau mỗi lần chạy trừ message keep; -1 = giữ toàn bộ; N = giữ N message), `log_output`, `config`.

### 3.2 Vòng đời thực thi 1 agent node (`runtime/node/executor/agent_executor.py`)

```
execute(node, inputs):
  provider_class = ProviderRegistry.get_provider(cfg.provider); client = provider.create_client()
  external_tool_specs = tool_manager.get_tool_specs(cfg.tooling)
  skill_manager = AgentSkillManager(allow=..., available_tool_names=...)
  conversation = [system(role + <available_skills> xml)] + inputs (PROMPT: gộp text / MESSAGES: giữ role)
  if thinking: _apply_pre_generation_thinking(...)           # stage PRE_GEN_THINKING
  _apply_memory_retrieval(...)                               # chèn memory vào conversation
  response = _invoke_provider(...)  (bọc tenacity Retrying theo AgentRetryConfig)
  if response.has_tool_calls(): _handle_tool_calls(...)      # vòng lặp tool đến khi hết tool_calls hoặc đạt loop limit
  _persist_message_attachments(...)                          # lưu ảnh/file model sinh ra vào attachment store
  if thinking: _apply_post_generation_thinking(...)          # stage POST_GEN_THINKING (self-reflection)
  _update_memory(...)                                        # ghi memory
  return [Message(role=ASSISTANT, content, metadata.source=node.id)]
except Exception: return [Message("Error calling model ...")]   # lỗi không làm vỡ graph
```

Các giai đoạn được mã hoá trong `entity/enums.py::AgentExecFlowStage = pre_gen_thinking | gen | post_gen_thinking | finished` và memory có thể cấu hình `retrieve_stage` theo từng giai đoạn.

### 3.3 Quản lý "đội ngũ"

2.0 **không có khái niệm company/roster**; "đội" đơn giản là tập node trong YAML. Prompt vai trò được nhúng trực tiếp vào `role:` của mỗi node (xem `ChatDev_v1.yaml`: "Chief Executive Officer", "Chief Product Officer", "Programmer Coding", "Code Reviewer", "Software Test Engineer" … mỗi vai một node, cùng một vai có thể xuất hiện nhiều node khác nhau theo phase).

Ngược lại 1.0 có `ChatEnv.recruit()` + `Roster` + `recruitments[]` trong `ChatChainConfig.json` (xem mục 10).

---

## 4. Giao tiếp giữa các agent (inter-agent communication)

### 4.1 Message model dùng chung (`entity/messages.py`)

```python
class MessageRole(str, Enum): SYSTEM, USER, ASSISTANT, TOOL
class MessageBlockType(str, Enum): TEXT, IMAGE, AUDIO, VIDEO, FILE, DATA
@dataclass class AttachmentRef: attachment_id, mime_type, name, size, sha256, local_path, remote_file_id, data_uri, metadata
@dataclass class MessageBlock: type, text, attachment: AttachmentRef | None ...
@dataclass class ToolCallPayload: id, function_name, arguments ; to_openai_dict()
@dataclass class Message: role, content: str | List[MessageBlock], metadata{source, context_trace,...}, tool_calls, tool_call_id, name, preserve_role
```
=> Multimodal, provider-agnostic; `metadata.source` luôn = node id tạo ra message.

### 4.2 Truyền message qua edge

Mỗi node có `input: List[Message]` (hàng đợi) và `output: List[Message]`. Sau khi node chạy, `GraphExecutor._execute_node()` lặp qua `node.iter_outgoing_edges()` và gọi `_process_edge_output(edge_link, output_msg, node)` → `condition_manager.process(...)`. Edge có các cờ (`entity/configs/edge/edge.py`):

| Trường | Ý nghĩa |
|---|---|
| `trigger` (true) | Edge tham gia topo sort; target đợi source xong. `false` = chỉ truyền data |
| `condition` | `'true'` / `{type: keyword, config:{any,none,regex,case_sensitive}}` / `{type: function, config:{name}}` |
| `carry_data` (true) | Có copy payload vào input của target không |
| `keep_message` (false) | Message được giữ lại trong input của target qua các lần clear (dùng cho task gốc USER → mọi agent) |
| `clear_context` / `clear_kept_context` | Xoá input của target trước khi nhận |
| `process` | Transform payload: `regex_extract` (mode replace_content/metadata/data_block, on_no_match) hoặc `function` |
| `dynamic` | Map/Tree fan-out (xem 8.5) |

Khi input gộp thành prompt (`NodeExecutor._inputs_to_text`):
```
=== INPUT FROM <source> (<role>) ===

<text>
```
Đây là "ngôn ngữ chung" giữa các agent: mọi thứ đều đi qua text/blocks của Message, **không có shared mutable state giữa agent** ngoài memory store (`blackboard` memory là cơ chế chia sẻ có chủ đích: "Shared blackboard memory allowing multiple nodes to read/write").

### 4.3 Agent với con người (human-in-loop)

Node `human` (`runtime/node/executor/human_executor.py`) gọi `HumanPromptService.request(node.id, description, inputs=flattened_text)`. `utils/human_prompt.py` định nghĩa `PromptChannel` Protocol với 2 hiện thực: `CliPromptChannel` (input()) và `server/services/prompt_channel.py::WebPromptChannel` (WS). Human node bị `ResourceManager` giới hạn `resource_limit=1` nên không bao giờ có 2 human node hỏi cùng lúc.

### 4.4 Agent đối thoại 2 chiều (1.0)

1.0 dùng `camel.agents.RolePlaying`: 2 `ChatAgent` (assistant & user) luân phiên; `Phase.chatting()` lặp `chat_turn_limit` vòng, dừng khi một bên trả lời có `<INFO>` (`ChatAgent.info`) → "seminar conclusion". 2.0 thay bằng cycle trong đồ thị + `keyword` condition `any: ["<INFO>"]` (xem `ChatDev_v1.yaml` dòng ~716-760).

---

## 5. Session / state management và persistence

### 5.1 Session store (in-memory)

`server/services/session_store.py`:
```python
class SessionStatus(Enum): IDLE, RUNNING, WAITING_FOR_INPUT, COMPLETED, ERROR, CANCELLED
@dataclass class WorkflowSession:
    session_id, yaml_file, task_prompt, task_attachments, status, created_at, updated_at
    executor, graph, current_node_id
    waiting_for_input, pending_input_data, human_input_future: Future, human_input_value
    results, error_message
    artifact_queue: ArtifactEventQueue
    cancel_event: threading.Event, cancel_reason
    message_buffer: list   # MAX_BUFFER_SIZE = 1000, replay khi reconnect
class WorkflowSessionStore:  _sessions: Dict[str, WorkflowSession]   # thuần RAM
```
- GC: `WebSocketManager._gc_loop()` mỗi giờ xoá session terminal quá `SESSION_TTL_SECONDS = 24h`.
- Reconnect: `WebSocketManager.connect(websocket, session_id)` → nếu session tồn tại: đóng socket cũ, **replay toàn bộ `message_buffer`** rồi gửi `session_resumed` snapshot. Frontend `LaunchView.vue` xử lý `session_resumed` để khôi phục `selectedFile`, trạng thái.
- Gửi từ worker thread: `send_message_sync()` dùng `asyncio.run_coroutine_threadsafe(..., self._owner_loop)` (fix lỗi "attached to a different loop").

### 5.2 Human input như một Future

`session_execution.py::SessionExecutionController`:
```python
def set_waiting_for_input(session_id, node_id, input_data): session.human_input_future = Future(); status = WAITING_FOR_INPUT
def wait_for_human_input(session_id, timeout=1800.0):   # poll 1s, kiểm tra cancel_event, raise TimeoutError
def provide_human_input(session_id, user_input): future.set_result(user_input)
```
WS message vào: `{"type":"human_input","data":{"input": "...", "attachments": [ids]}}`; ra: `{"type":"human_input_required","data":{node_id,input,task_description}}`.

### 5.3 Persistence trên đĩa

- Thư mục workflow: `WareHouse/session_<session_id>/` (hoặc `WareHouse/<name>/` khi chạy CLI/SDK). `GraphContext.directory` = `output_root/name`; `code_workspace/` cho file code, `code_workspace/attachments/` cho `AttachmentStore`.
- Sau khi chạy: `ResultArchiver.export()` ghi `token_usage_<name>.json`, `execution_logs.json`; memory `simple` ghi `memory_<store>.json`.
- `GET /api/sessions/{id}/download` zip toàn bộ thư mục (từ chối symlink – fix bảo mật gần đây, commit `1970de9`).
- **Không có checkpoint/resume giữa chừng**: nếu server restart thì session mất; chỉ có reconnect WS trong cùng tiến trình.
- Vị trí node trên canvas lưu file JSON riêng (`vuegraphs_storage.py`), YAML lưu `yaml_instance/`.

### 5.4 Cancel

`WorkflowRunService.request_cancel()` set `session.cancel_event` + `executor.request_cancel()`; mọi `NodeExecutor._ensure_not_cancelled()` và `GraphExecutor._raise_if_cancelled()` ném `WorkflowCancelledError`. Cancel chỉ có hiệu lực tại ranh giới node/tool call (không kill LLM request đang chạy).

---

## 6. Agent thực thi terminal / file như thế nào

### 6.1 Cơ chế opt-in toàn cục

`utils/code_execution.py`:
```python
UNSAFE_CODE_EXECUTION_ENV = "MAC_ALLOW_UNSAFE_CODE_EXECUTION"
def require_unsafe_code_execution(operation):
    # Never consult workflow variables, tool arguments, or child-process env overrides.
    if os.environ.get(...).lower() not in {"1","true","yes","on"}: raise SecurityError(...)
```
Mặc định **tắt**; `.env.example` ghi rõ "this is not a sandbox". Áp dụng cho python node, `execute_code`, `uv_*`, Manim, `mcp_local`.

### 6.2 Node `python` (`runtime/node/executor/python_executor.py`)

Trích code block ```` ```python ```` từ message cuối, ghi `code_workspace/<node_id>[_run-N].py`, chạy `subprocess.run([interpreter, *args, script], cwd=workspace, env={..., MAC_CODE_WORKSPACE, MAC_CODE_SCRIPT, MAC_NODE_ID}, timeout=timeout_seconds)` → stdout thành message ASSISTANT; lỗi → failure message (không throw).

### 6.3 Function-calling tools cho agent (`functions/function_calling/`)

| File | Hàm | Ghi chú |
|---|---|---|
| `file.py` (1100 dòng) | `describe_available_files, list_directory, create_folder, delete_path, load_file, save_file, read_text_file_snippet, read_file_segment, apply_text_edits, rename_path, copy_path, move_path, search_in_files` | Nhận `_context` (attachment_store, python_workspace_root, graph_directory); `FileToolContext.resolve_under_workspace()` chặn path traversal |
| `code_executor.py` | `execute_code(code, time_out)` | ghi file tạm rồi `subprocess.run(sys.executable, ...)` |
| `uv_related.py` | `install_python_packages, init_python_env, uv_run` | chạy `uv` CLI trong workspace, có timeout, validate flag/args/env |
| `web.py` | `web_search` (ddgs/serper), `get_webpage_content` (jina) | |
| `deep_research.py`, `video.py`, `weather.py`, `user.py` (`call_user`) | | |

Tool được nạp động bởi `utils/function_manager.py::FunctionManager.load_functions()` (import từng `.py`, lấy mọi hàm public) và mô tả schema tự động từ type hints/docstring (`utils/function_catalog.py::ParamMeta`). Cú pháp trong YAML: `- name: uv_related:All` (tất cả hàm của module) hoặc `- name: save_file`.

### 6.4 Nhận diện artifact

`workflow/hooks/workspace_artifact.py::WorkspaceArtifactHook` chụp snapshot (sha256, size) thư mục workspace trước/sau mỗi node `python`/`agent` (max 500 file / 500 MB), file mới/đổi → `AttachmentRecord` → WS `artifact_created` → frontend hiển thị/preview, tải qua `/api/sessions/{id}/artifacts/{artifact_id}`.

### 6.5 So sánh 1.0

1.0 `ChatEnv.exist_bugs()` chạy `python main.py` trong thư mục dự án bằng `subprocess.Popen(shell=True)`, ngủ 3 giây rồi SIGTERM; `Codes._rewrite_codes()` parse code block ```` FILENAME\n```LANGUAGE ```` từ output LLM và ghi file; `git_management` tạo commit mỗi phase. Hoàn toàn không sandbox.

---

## 7. Tích hợp AI provider

### 7.1 Abstraction (`runtime/node/agent/providers/base.py`)

```python
class ModelProvider(ABC):
    def __init__(self, config: AgentConfig): base_url, api_key, model_name, params
    @abstractmethod def create_client(self)
    @abstractmethod def call_model(self, client, conversation: List[Message], timeline: List[Any], tool_specs=None, **kwargs) -> ModelResponse
    @abstractmethod def extract_token_usage(self, response) -> TokenUsage
class ProviderRegistry: register(name, cls, label, summary) / get_provider / list_providers / iter_metadata
```
`builtin_providers.py` đăng ký `openai` (bắt buộc) và `gemini` (optional, try/except ImportError).

### 7.2 OpenAIProvider (809 dòng)

- Hỗ trợ cả **Responses API** (mặc định nếu client có `.responses`) lẫn **Chat Completions** (`params.protocol: chat`), fallback sang chat khi responses lỗi. Vì vậy mọi endpoint OpenAI-compatible (Ollama, LM Studio, vLLM, DeepSeek…) dùng được qua `BASE_URL`.
- Serialize `MessageBlock` → content parts (image/file inline base64, CSV/text inline ≤200k ký tự), parse `tool_call`, `image_generation_call`, strip `<think>` tokens.
- Token usage: `_track_token_usage()` → `TokenTracker.record_usage(node_id, model_name, usage, provider)`; export `token_usage_<name>.json` và trả về trong `workflow_completed`.

### 7.3 Cấu hình

- Env: `BASE_URL`, `API_KEY` (dùng chung cho mọi provider), `MAC_ALLOW_UNSAFE_CODE_EXECUTION`, `SERPER_DEV_API_KEY`, `JINA_API_KEY`.
- YAML: `${VAR}` được resolve bởi `entity/config_loader.py::prepare_design_mapping` + `utils/vars_resolver.py` (từ `vars:` ở root rồi env).
- Retry (`AgentRetryConfig`): tenacity `Retrying(stop_after_attempt(max_attempts), wait_random_exponential(min,max), retry_if_exception(should_retry))` với `retry_on_status_codes` (429/5xx), `retry_on_exception_types`, `retry_on_error_substrings`.
- Embedding cho memory: `EmbeddingConfig(provider=openai, model=text-embedding-3-small)`.

### 7.4 1.0

`camel/model_backend.py::OpenAIModel.run()` – chỉ `client.chat.completions.create`, tự đếm token bằng tiktoken để set `max_tokens`, `ModelType` enum cứng (gpt-3.5/4/4o). Không có tool calling.

---

## 8. Workflow engine (2.0)

### 8.1 YAML schema (`yaml_template/design.yaml`, `docs/user_guide/en/field_specs.md`)

```yaml
version: "0.4.0"
vars: { KEY: value }           # chỉ ở root; dùng ${KEY} trong graph
graph:
  id: <str>                    # [A-Za-z0-9_-]
  description, log_level (DEBUG), is_majority_voting (false), initial_instruction
  start: [node_id,...]         # entry nodes nhận task prompt
  end: [node_id,...]           # thứ tự ưu tiên lấy output cuối
  nodes:
    - id, type: agent|human|subgraph|python|passthrough|literal|loop_counter|loop_timer
      description, context_window (0), log_output (true)
      config: <tuỳ type>       # agent: AgentConfig; human: {description}; literal: {content, role}
                               # subgraph: {type: config|file, config: {...}|{path}}; python: {interpreter,args,env,timeout_seconds}
                               # loop_counter: {max_iterations, reset_on_emit, message}; loop_timer: {...}
  edges:
    - from, to, trigger, condition, carry_data, keep_message, clear_context, clear_kept_context, process, dynamic
  memory:
    - name, type: simple|file|blackboard|mem0, config: {...}
```
Validation 3 lớp: `check/check_yaml.py::validate_design` (schema), `DesignConfig.from_dict` (typed dataclass, ConfigError có path), `check/check_workflow.py::check_workflow_structure` (logic: node tồn tại, edge hợp lệ…). `make validate-yamls` chạy `tools/validate_all_yamls.py`.

### 8.2 Node types

| Type | Executor | Vai trò |
|---|---|---|
| `agent` | AgentNodeExecutor | LLM + tools + memory + thinking + skills |
| `human` | HumanNodeExecutor | Dừng chờ người (serialized, limit 1) |
| `subgraph` | SubgraphNodeExecutor | Nhúng graph con (inline hoặc file), có `end` riêng |
| `python` | PythonNodeExecutor | Chạy code block Python |
| `passthrough` | PassthroughNodeExecutor | Gom/chuyển tiếp input (`only_last_message`) – dùng làm "join"/"USER"/"FINAL" |
| `literal` | LiteralNodeExecutor | Phát một message cố định (dùng làm phase prompt) |
| `loop_counter` | LoopCounterNodeExecutor | Chặn edge ra cho tới khi đạt N lần → phát message (break loop) |
| `loop_timer` | LoopTimerNodeExecutor | Tương tự theo thời gian |

### 8.3 Mô hình thực thi (`workflow/graph.py::GraphExecutor.run`)

```python
graph_manager.build_graph()           # predecessors/successors, layers (topo), detect cycles
self._prepare_edge_conditions()       # compile condition + processor cho từng EdgeLink
self._build_memories_and_thinking()   # memory stores, thinking managers, node executors
for node in start_nodes: node.start_triggered = True; node.append_input(task messages)
if is_majority_voting: MajorityVoteStrategy   # chạy tất cả node song song, Counter(output).most_common(1)
elif has_cycles:       CycleExecutionStrategy # Tarjan SCC → super node → recursive
else:                  DagExecutionStrategy   # layers → ParallelExecutor (ThreadPoolExecutor max_workers=len(layer))
self._collect_all_outputs(); final = get_final_output(); _save_memories(); ResultArchiver.export()
```

- **Parallelism**: node cùng layer chạy song song bằng thread (`workflow/executor/parallel_executor.py`), node "blocking" (human) tách ra chạy tuần tự. Giới hạn tài nguyên theo loại node qua `ResourceManager` (semaphore theo `NodeCapabilities.resource_key/limit`).
- **Trigger semantics**: node chỉ chạy khi có ít nhất một edge `trigger=true` thoả condition đã "triggered" (hoặc là start node). Sau khi chạy: `node.reset_triggers()`, `node.clear_input(preserve_kept=True, context_window=...)`.
- **Cycles** (`docs/user_guide/en/execution_logic.md`): Tarjan tìm SCC; mỗi SCC → super node; trong SCC xác định **initial node** duy nhất (được trigger từ ngoài); mỗi vòng lặp xoá logic các edge vào initial node, detect nested cycle đệ quy, chạy theo layer; điều kiện thoát: exit edge triggered, `max_iterations` (mặc định 100), `loop_timer`, hoặc initial node không được re-trigger.
- **Context trace**: với `context_window != 0`, output có `metadata.context_trace` để khôi phục lịch sử hội thoại của chính node đó qua các iteration (`_restore_context_trace`, pseudo self-edge).

### 8.4 Checkpoint / retry / human-in-loop

| Khả năng | Trạng thái trong ChatDev 2.0 |
|---|---|
| Retry LLM call | Có, tenacity per-node (`AgentRetryConfig`) |
| Retry node / re-run từ node | **Không** |
| Checkpoint / resume sau crash | **Không** (session RAM; chỉ replay WS buffer) |
| Human-in-loop | Có: node `human`, tool `call_user`, timeout 1800s, attachments kèm |
| Cancel | Có, cooperative qua `threading.Event` |
| Timeout tool | `tooling.config.timeout` (function), `timeout` (mcp) |
| Majority voting | Có (`is_majority_voting`) |
| Batch run | `POST /api/workflows/batch` + `batch_run_service.py` (nhiều task prompt, WS `batch_*` events) |
| Sync run / SSE | `POST /api/workflow/run` (Accept: text/event-stream → stream log) ; Python SDK `runtime/sdk.py::run_workflow()` |

### 8.5 Dynamic edges (Map / Tree)

`entity/configs/edge/dynamic_edge_config.py` + `workflow/executor/dynamic_edge_executor.py`: edge có `dynamic: {type: map|tree, split: {type: message|regex|json_path}, config: {max_parallel, group_size}}` → target node được "nhân bản ảo" chạy song song cho từng unit (ThreadPool), `tree` giảm dần theo nhóm `group_size` đến 1 output. Edge tĩnh đi vào cùng node được replicate cho mọi instance. Đây là cơ chế "scatter-gather" đáng tham khảo.

### 8.6 Memory & Thinking

- Memory stores (`runtime/node/agent/memory/builtin_stores.py`): `simple` (JSON + embedding), `file` (index tài liệu, FAISS), `blackboard` (shared, max_items), `mem0`. Gắn vào agent bằng `memories: [{name, retrieve_stage[], top_k, similarity_threshold, read, write}]`.
- Thinking: `reflection` (`SelfReflectionThinkingManager`, `reflection_prompt`) – LLM tự phê bình output rồi sửa (tương đương `need_reflect` của 1.0 nhưng trong cùng node).

---

## 9. Skill / plugin / tool / MCP

### 9.1 Ba loại tooling (`entity/configs/node/tooling.py`)

```yaml
tooling:
  - type: function
    prefix: ''                      # tránh trùng tên tool
    config:
      tools: [{name: save_file}, {name: uv_related:All}, {name: x, description, parameters, auto_fill}]
      auto_load: true
      timeout: null
  - type: mcp_remote
    config: {server: https://host/mcp, headers: {Authorization: Bearer ${TOKEN}}, timeout: 15, cache_ttl: 0, tool_sources: []}
  - type: mcp_local
    config: {command: uvx, args: [blender-mcp], cwd, env, inherit_env: true, startup_timeout: 10, wait_for_log: "MCP ready", cache_ttl}
```

`runtime/node/agent/tool/tool_manager.py::ToolManager`:
- `get_tool_specs(tool_configs) -> List[ToolSpec]` (gộp 3 nguồn, kiểm tra trùng tên, gắn `metadata._config_index/original_name`).
- `execute_tool()` → `_execute_function_tool` (inject `_context` nếu hàm khai báo), `_execute_mcp_remote_tool` (fastmcp `Client(StreamableHttpTransport)`, retry 3 lần backoff), `_execute_mcp_local_tool` (`_StdioClientWrapper` giữ process stdio sống trong thread riêng + event loop riêng).
- Kết quả MCP (`mcp.types.TextContent/ImageContent/EmbeddedResource`) → `MessageBlock`, binary → ghi vào `AttachmentStore`.
- `ToolSpec.to_openai_dict()` / `to_gemini_function()` – mapping sang schema provider.

### 9.2 Agent Skills (`runtime/node/agent/skills/manager.py`)

- Thư mục cố định `.agents/skills/<name>/SKILL.md`, frontmatter YAML bắt buộc `name` (= tên thư mục), `description`; tuỳ chọn `allowed-tools`, `compatibility`. Giới hạn file 128 KB.
- Khi `skills.enabled: true`, system prompt được nối `<available_skills><skill><name/><description/><location/><allowed_tools/></skill></available_skills>` và agent có 2 tool nội bộ `activate_skill(skill_name)` (trả về toàn bộ SKILL.md) và `read_skill_file(skill_name, relative_path)` (chặn path traversal). Skill chỉ được lọc nếu `allowed-tools` của nó có trong tool của agent (`_is_skill_compatible`).
- Đây là format tương thích với Claude Code / Agent Skills chuẩn – **tái dùng trực tiếp được**.

### 9.3 Plugin registry

`utils/registry.py::Registry` + `schema_registry/registry.py`: mọi thứ (node type, provider, memory store, thinking mode, edge condition, edge processor, subgraph source) đều `register_*()`; `runtime/bootstrap/schema.py::ensure_schema_registry_populated()` import 5 module builtin một lần. Schema được export cho UI qua `server/config_schema_router.py` (`--inspect-schema` trong `run.py`). Pattern này giúp frontend form tự sinh – nên giữ trong bản TypeScript (Zod schema → JSON Schema → form).

---

## 10. ChatDev 1.0 vs 2.0

### 10.1 Mô hình 1.0 (Virtual Software Company)

```
run.py --task --name --org --config Default|Art|Human|Incremental --model GPT_4O
  → ChatChain(ChatChainConfig.json, PhaseConfig.json, RoleConfig.json)
     .pre_processing()        # dọn WareHouse, ghi meta
     .make_recruitment()      # ChatEnv.recruit(role) cho mỗi recruitments[]
     .execute_chain()         # for phase_item in chain: execute_step()
        SimplePhase  → phases[name].execute(chat_env, max_turn_step, need_reflect)
        ComposedPhase→ CodeCompleteAll/CodeReview/Test/Art/HumanAgentInteraction.execute(chat_env)
                        for cycle in 1..cycleNum: for sub in Composition: break_cycle()? execute
     .post_processing()       # copy code, log, thống kê
```

`CompanyConfig/Default/ChatChainConfig.json` (trích):
```json
{"chain":[
  {"phase":"DemandAnalysis","phaseType":"SimplePhase","max_turn_step":-1,"need_reflect":"True"},
  {"phase":"LanguageChoose","phaseType":"SimplePhase","max_turn_step":-1,"need_reflect":"True"},
  {"phase":"Coding","phaseType":"SimplePhase","max_turn_step":1,"need_reflect":"False"},
  {"phase":"CodeCompleteAll","phaseType":"ComposedPhase","cycleNum":10,"Composition":[{"phase":"CodeComplete",...}]},
  {"phase":"CodeReview","phaseType":"ComposedPhase","cycleNum":3,"Composition":[{"phase":"CodeReviewComment"},{"phase":"CodeReviewModification"}]},
  {"phase":"Test","phaseType":"ComposedPhase","cycleNum":3,"Composition":[{"phase":"TestErrorSummary"},{"phase":"TestModification"}]},
  {"phase":"EnvironmentDoc","phaseType":"SimplePhase","need_reflect":"True"},
  {"phase":"Manual","phaseType":"SimplePhase"}],
 "recruitments":["Chief Executive Officer","Counselor","Chief Human Resource Officer","Chief Product Officer","Chief Technology Officer","Programmer","Code Reviewer","Software Test Engineer","Chief Creative Officer"],
 "clear_structure":"True","gui_design":"True","git_management":"False","web_spider":"False","self_improve":"False","incremental_develop":"False","with_memory":"False",
 "background_prompt":"ChatDev is a software company powered by multiple intelligent agents..."}
```

`PhaseConfig.json`: mỗi phase = `{assistant_role_name, user_role_name, phase_prompt[]}` với placeholder `{task} {modality} {language} {ideas} {codes} {unimplemented_file} {test_reports} {error_summary} {gui}`; điều kiện dừng thảo luận là một dòng bắt đầu `<INFO>`.

`RoleConfig.json`: 9 vai, mỗi vai là mảng dòng prompt bắt đầu bằng `{chatdev_prompt}` (= background_prompt) + mô tả trách nhiệm + `Here is a new customer's task: {task}.`

`chatdev/phase.py::Phase.execute()`:
```python
self.update_phase_env(chat_env)           # kéo biến từ ChatEnv.env_dict vào placeholders
self.seminar_conclusion = self.chatting(... assistant_role, user_role, phase_prompt, chat_turn_limit, need_reflect ...)
chat_env = self.update_chat_env(chat_env) # ghi kết luận ngược vào env_dict (modality/language/codes/...)
```
`chatting()` = `RolePlaying.step()` lặp tới khi `<INFO>`; `self_reflection()` = một phiên CEO↔Counselor đọc toàn bộ transcript để chốt kết luận.

`ChatEnv.env_dict`: `task_prompt, modality, ideas, language, review_comments, error_summary, test_reports, directory, codes (Codes), requirements, manuals`. Codes được parse từ markdown block và ghi vào `WareHouse/<name>_<org>_<timestamp>/`; `exist_bugs()` chạy `python main.py` 3s; `generate_images_from_codes()` gọi DALL-E cho Art mode; `git_management` commit mỗi phase; `incremental_develop` nạp code có sẵn (`--path`).

Visualizer 1.0: Flask `visualizer/app.py` với `/`, `/chain_visualizer`, `/replay`, `/get_messages`, `/send_message` – replay log, avatar theo role (`visualizer/static/figures/{ceo,cpo,cto,programmer,reviewer,tester,designer}.png`).

### 10.2 Bảng so sánh

| Khía cạnh | ChatDev 1.0 | ChatDev 2.0 (DevAll) |
|---|---|---|
| Mô hình | Chuỗi phase tuyến tính (ChatChain) với ComposedPhase lặp cố định `cycleNum` | Đồ thị YAML tổng quát (DAG + cycle + subgraph + dynamic map/tree) |
| Đơn vị | Phase = 1 cuộc đối thoại 2 vai (instructor ↔ assistant) | Node = 1 lần gọi LLM (hoặc human/python/literal…), đối thoại = chu trình trong graph |
| Cấu hình | 3 JSON (chain/phase/role) + class Python cho mỗi phase (`chatdev/phase.py`) | 1 YAML tự mô tả, không cần code cho workflow mới; schema registry + UI form |
| Vai trò | Roster/recruitment, role prompt tập trung | Role nhúng trong từng node; không có khái niệm company |
| State chung | `ChatEnv.env_dict` mutable dùng chung | Message queue trên edge + memory store (blackboard) |
| Dừng vòng lặp | `break_cycle()` code hoá (vd `<INFO> Finished`, `exist_bugs_flag`) | `keyword`/`function` condition + `loop_counter`/`loop_timer` |
| Tool | Không (LLM output code block → parse) | Function calling + MCP remote/local + Agent Skills |
| Model | OpenAI chat only, enum cứng | Provider registry (OpenAI responses/chat, Gemini), base_url bất kỳ |
| Human | Config "Human": phase `CodeReviewHuman` input() | Node `human` + WS prompt channel + attachments + `call_user` tool |
| Code exec | `python main.py` 3s, shell=True | python node / execute_code / uv_run, opt-in env flag, vẫn không sandbox |
| Frontend | Flask log viewer + replay | Vue 3 + Vue Flow builder + launch console + batch run |
| API | Không | REST + WS + SSE + Python SDK |
| Persistence | WareHouse + log file | WareHouse + JSON logs/token; session RAM |
| Memory | `with_memory` (ECL experience) | simple/file/blackboard/mem0 + embedding |
| Thinking | `need_reflect` (CEO↔Counselor) | `thinking.reflection` trong node |

### 10.3 ChatDev_v1.yaml – cách 2.0 tái hiện 1.0

`yaml_instance/ChatDev_v1.yaml` gồm: node `USER` (passthrough nhận task, edge `trigger:false, keep_message:true` tới mọi agent để task luôn trong context), các `literal` node chứa phase prompt ("Coding Phase Prompt for Assistant", "Code Review Comment Phase Prompt…", "Test Error Summary…", "Manual Phase Prompt…"), agent node theo vai (CEO, CPO, Programmer Coding/Code Complete/Code Review/Test Modification/Test Error Summary, Code Reviewer, Software Test Engineer), `loop_counter` cho CodeCompleteAll(5)/CodeReview/Test(3)/TestModification(5)/Manual, node `PSEUDO` gom các nhánh thoát bằng `condition: {type: keyword, config: {any: ["<INFO>"]}}`, node `FINAL`. Agent dùng tools `uv_related:All, apply_text_edits, create_folder, describe_available_files, delete_path, read_file_segment, search_in_files, save_file, list_directory` thay vì parse code block. `${COMMON_PROMPT}` = background prompt 1.0.

---

## 11. Khả năng tái sử dụng cho AI WORKSPACE (TypeScript/NestJS + React + Phaser + Claude Code/Codex CLI + PostgreSQL + Docker)

### 11.1 Tái sử dụng trực tiếp (copy/port nội dung, không phải code)

1. **Role prompts & phase prompts 1.0** (`v1.1.6:CompanyConfig/Default/RoleConfig.json`, `PhaseConfig.json`) và `background_prompt` – dùng làm bộ "nhân sự mặc định" của công ty ảo (CEO/CPO/CTO/Programmer/Reviewer/Tester/Designer/Counselor/CHRO) và template phase (DemandAnalysis, LanguageChoose, Coding, CodeComplete, CodeReviewComment/Modification, TestErrorSummary/Modification, EnvironmentDoc, Manual). Cần sửa placeholder và bỏ ràng buộc "code block format" vì agent của ta có tool thật (Claude Code/Codex CLI).
2. **Chuỗi phase mặc định** (`ChatChainConfig.json`) + `ChatDev_v1.yaml` – làm pipeline "Software Company" mặc định; map ComposedPhase/cycleNum sang loop node.
3. **WS event protocol**: `connection`, `session_resumed`, `workflow_started`, `log{level,node_id,event_type,details,duration}`, `human_input_required`, `input_received`, `artifact_created`, `workflow_completed{results,summary,token_usage}`, `workflow_cancelled`, `error`, `status`, `batch_*`; inbound `human_input`, `ping`, `get_status`, `cancel`. Có thể dùng y nguyên làm Socket.IO/WS contract của NestJS Gateway.
4. **`EventType` enum** (`entity/enums.py`): NODE_START/END, EDGE_PROCESS, MODEL_CALL, TOOL_CALL, AGENT_CALL, HUMAN_INTERACTION, THINKING_PROCESS, MEMORY_OPERATION, WORKFLOW_START/END – trực tiếp thành bảng `workflow_events` trong PostgreSQL và nguồn animation cho Phaser (agent "đi tới bàn" khi NODE_START, "gõ phím" khi MODEL_CALL…).
5. **SKILL.md format** + `AgentSkillManager` semantics (`activate_skill`/`read_skill_file`, `allowed-tools`) – trùng chuẩn Claude Code Skills nên chỉ cần trỏ thư mục `.agents/skills` / `.claude/skills`.
6. **YAML workflow schema** (field names: `nodes/edges/start/end/vars`, edge `trigger/condition/carry_data/keep_message/clear_context/process/dynamic`) – chuyển thẳng sang Zod schema; giữ tương thích để import các YAML mẫu trong `yaml_instance/`.
7. **Sprite pixel-art 12 nhân vật** (`frontend/public/sprites/`) & ý tưởng gán nhân vật theo node_id + walking animation khi active – kiểm tra license trước khi dùng cho Phaser.
8. **Tài liệu** `docs/user_guide/en/*.md` (execution_logic, dynamic_execution, field_specs, mcp, nodes/*) – làm spec tham chiếu khi viết engine TS.
9. **Test case bảo mật** (`tests/test_session_workspace_security.py`, `test_attachment_boundary_security.py`, `test_code_execution_security.py`) – port thành Jest tests cho path traversal/symlink/attachment.

### 11.2 Tham chiếu kiến trúc (viết lại bằng TS theo cùng thiết kế)

- `GraphExecutor` + `ExecutionStrategy` (DAG / Cycle / MajorityVote) + `ParallelExecutor` + `ResourceManager` (semaphore theo node type) → TS class với `Promise.all` + p-limit; human node limit 1.
- `CycleManager` (Tarjan SCC) + `TopologyBuilder` (super-node) + `CycleExecutor` (initial node, nested cycle, exit conditions) → thuật toán giữ nguyên; thay thread bằng async.
- `DynamicEdgeExecutor` (Map/Tree, split message/regex/json_path) → scatter-gather cho review song song nhiều file.
- `NodeExecutor`/`ExecutionContext` (tool_manager, log_manager, memory_managers, token_tracker, global_state, human_prompt_service, cancel_event) → NestJS provider injection.
- `HumanPromptService` + `PromptChannel` Protocol (CLI vs Web) + `SessionExecutionController` (Future + poll + timeout + cancel) → TS `Deferred` + Redis/PG-backed pending prompt để sống sót khi restart.
- `WebSocketManager` reconnect + replay buffer + `session_resumed` snapshot → giữ ý tưởng nhưng lưu buffer trong PostgreSQL/Redis.
- `WorkspaceArtifactHook` (snapshot sha256 trước/sau node) → tương đương `git status`/chokidar trong Docker sandbox để phát `artifact_created`.
- `TokenTracker` (per node/model/provider, export JSON) → bảng `token_usage`.
- Registry pattern + `FIELD_SPECS` → schema export cho React form tự sinh (JSON Schema + react-jsonschema-form hoặc tự viết).
- `AgentRetryConfig` semantics (status codes, exception types, substrings) → cấu hình retry cho adapter Claude/Codex CLI.
- `check_yaml` / `check_workflow_structure` 3-lớp validation.
- 1.0 `Phase.execute = update_phase_env → chatting → update_chat_env` + `ComposedPhase.break_cycle` → mô hình "phase as first-class" nếu muốn giữ UX "công ty ảo" đơn giản hơn graph tổng quát (gợi ý: expose cả 2: Company pipeline template sinh ra graph).

### 11.3 Phải viết mới / thay thế hoàn toàn

| Thành phần ChatDev | Lý do không dùng được | Hướng cho AI WORKSPACE |
|---|---|---|
| Toàn bộ Python runtime (workflow/, runtime/, entity/, server/) | Ngôn ngữ khác; đồng bộ thread, không có DB | NestJS modules: WorkflowModule, SessionModule, AgentModule, ToolModule |
| `ModelProvider` OpenAI/Gemini | Agent brain của ta là **Claude Code / Codex CLI** (agentic, tự có tool/file/terminal) chứ không phải raw chat API | `AgentBrainAdapter` interface: `run(task, workspace, opts) → stream events`; adapter `ClaudeCodeAdapter` (spawn `claude -p --output-format stream-json`), `CodexAdapter` (`codex exec --json`); map stream events → `EventType` |
| `ToolManager` function-calling + file tools Python | Claude Code/Codex đã có Read/Edit/Bash/MCP; không cần tái hiện tool loop | Chỉ cần MCP config generator (`.mcp.json`) + skill dir mount vào sandbox |
| `python` node / `execute_code` / `uv_run` (subprocess, không sandbox) | Yêu cầu Docker sandbox | Node `shell`/`task` chạy trong container per-session (dockerode), workspace volume |
| `WorkflowSessionStore` in-memory, `WareHouse/` | Cần persistence, multi-instance | PostgreSQL (sessions, runs, node_runs, events, artifacts, token_usage), object storage cho artifact |
| Vue 3 + Vue Flow UI | Stack React | React + React Flow (API gần như 1:1 với Vue Flow) cho builder; Phaser cho office |
| Flask visualizer 1.0 | Lỗi thời | Phaser scene nhận `EventType` stream |
| `camel/` RolePlaying 2-agent loop | Gắn với chat API | Nếu cần "seminar" 2 agent: mô hình hoá bằng 2 node + cycle + keyword `<INFO>` như ChatDev_v1.yaml |
| Memory (faiss/mem0) | Python libs | pgvector trong PostgreSQL; giữ interface `retrieve(stage, query, top_k)` / `write()` |
| Checkpoint/resume | Không có trong ChatDev | Thiết kế mới: persist `node.input/output/triggers` sau mỗi node; resume từ node fail |

### 11.4 Khuyến nghị thứ tự port

1. Zod schema cho YAML (từ `design.yaml`) + validator 3 lớp; import được `yaml_instance/*.yaml`.
2. Engine lõi: GraphManager (topo layers) → DAG strategy → edge condition keyword/function → loop_counter/cycle (Tarjan) → subgraph → dynamic map/tree.
3. Node executors: `agent` (qua ClaudeCode/Codex adapter), `human`, `literal`, `passthrough`, `loop_counter`, `shell` (Docker).
4. Session/WS gateway theo protocol 2.0 + persistence PG + replay.
5. Company template: port `RoleConfig`/`PhaseConfig`/`ChatChainConfig` thành seed data + generator sinh graph (tham khảo `ChatDev_v1.yaml`).
6. React Flow builder với form tự sinh từ schema; Phaser office map `EventType` → animation, sprite theo agent.

---

## 12. Bảng tổng hợp tái sử dụng

| Module | Path | Tái sử dụng trực tiếp | Cần chỉnh sửa | Viết mới | Ghi chú |
|---|---|---|---|---|---|
| Role prompts 1.0 | `git show v1.1.6:CompanyConfig/Default/RoleConfig.json` | ✔ (nội dung prompt) | Placeholder `{task}`, bỏ yêu cầu code-block | | 9 vai CEO/CPO/CTO/CHRO/Counselor/Programmer/Reviewer/Tester/CCO |
| Phase prompts 1.0 | `git show v1.1.6:CompanyConfig/Default/PhaseConfig.json` | ✔ | Thay "Source Codes: {codes}" bằng "dùng tool đọc workspace" | | 13 phase, điều kiện `<INFO>` |
| Chain config 1.0 | `git show v1.1.6:CompanyConfig/*/ChatChainConfig.json` | ✔ (làm template Default/Art/Human/Incremental) | Map ComposedPhase/cycleNum → loop node | | |
| ChatChain/Phase/ChatEnv | `v1.1.6:chatdev/{chat_chain,phase,composed_phase,chat_env}.py` | | | ✔ | Chỉ tham chiếu mô hình `update_phase_env → chatting → update_chat_env`, `break_cycle` |
| CAMEL RolePlaying | `v1.1.6:camel/agents/role_playing.py` | | | ✔ | Thay bằng 2 node + cycle |
| Workflow mẫu port 1.0 | `yaml_instance/ChatDev_v1.yaml` | ✔ (import vào builder) | Đổi provider/tool names sang adapter mới | | 1021 dòng, loop_counter + keyword `<INFO>` |
| YAML schema | `yaml_template/design.yaml`, `entity/configs/**` | ✔ (field names) | | ✔ Zod/TS types | Giữ tương thích để import `yaml_instance/*` |
| Validation | `check/{check,check_yaml,check_workflow}.py` | | | ✔ | Port logic 3 lớp |
| GraphExecutor + strategies | `workflow/graph.py`, `workflow/runtime/execution_strategy.py` | | | ✔ | Tham chiếu vòng đời run(), trigger/reset/clear_input |
| Cycle engine | `workflow/cycle_manager.py`, `topology_builder.py`, `executor/cycle_executor.py` | | | ✔ | Giữ nguyên thuật toán Tarjan/super-node/initial node |
| Parallel + Resource | `workflow/executor/{parallel_executor,resource_manager}.py` | | | ✔ | p-limit + semaphore theo node type |
| Dynamic Map/Tree | `workflow/executor/dynamic_edge_executor.py`, `runtime/node/splitter.py` | | | ✔ | Scatter-gather |
| Edge conditions/processors | `runtime/edge/**`, `functions/edge/**` | ✔ (semantics keyword any/none/regex) | | ✔ | function condition → JS/TS plugin |
| Node registry pattern | `runtime/node/registry.py`, `schema_registry/registry.py`, `utils/registry.py` | | | ✔ | Decorator/DI trong NestJS + JSON Schema export |
| AgentNodeExecutor | `runtime/node/executor/agent_executor.py` | | | ✔ | Thay bằng ClaudeCode/Codex adapter; giữ stage pre/gen/post + memory hooks |
| Provider layer | `runtime/node/agent/providers/*` | | | ✔ | Interface `create_client/call_model/extract_token_usage` → `AgentBrainAdapter` |
| Retry policy | `entity/configs/node/agent.py::AgentRetryConfig` | ✔ (semantics) | | ✔ | p-retry |
| ToolManager / MCP | `runtime/node/agent/tool/tool_manager.py`, `entity/configs/node/tooling.py` | ✔ (schema mcp_remote/mcp_local) | | ✔ | Sinh `.mcp.json` cho Claude Code / `config.toml` cho Codex |
| Function tools | `functions/function_calling/*.py` | | | ✔ (hầu hết không cần) | Claude Code/Codex đã có file/shell tools |
| Agent Skills | `runtime/node/agent/skills/manager.py`, `.agents/skills/**` | ✔ (format SKILL.md, 3 skill mẫu) | | ✔ loader TS (nếu cần ngoài Claude Code) | Chuẩn trùng Claude Code Skills |
| Memory stores | `runtime/node/agent/memory/*` | | | ✔ | Interface giữ, backend pgvector |
| Thinking/reflection | `runtime/node/agent/thinking/*` | ✔ (prompt pattern) | | ✔ | |
| Human-in-loop | `utils/human_prompt.py`, `server/services/{session_execution,prompt_channel}.py`, `human_executor.py` | ✔ (protocol, timeout 1800s) | | ✔ | Pending prompt persist DB |
| Session store | `server/services/session_store.py` | ✔ (SessionStatus enum, snapshot fields) | | ✔ | PostgreSQL |
| WebSocket manager/protocol | `server/services/{websocket_manager,message_handler,websocket_logger}.py`, `docs/user_guide/en/ws_frontend_logic.md` | ✔ (message types, reconnect/replay semantics) | | ✔ NestJS Gateway | |
| REST API | `server/routes/*.py` | ✔ (endpoint list làm spec) | | ✔ | workflows CRUD, uploads, artifacts, sessions download, batch, run sync/SSE, tools |
| Logging / EventType | `utils/{logger,log_manager,structured_logger}.py`, `entity/enums.py` | ✔ (EventType, LogEntry shape) | | ✔ | Nguồn sự kiện cho Phaser |
| Token tracking | `utils/token_tracker.py` | ✔ (data model) | | ✔ | Bảng token_usage |
| Artifact hook | `workflow/hooks/workspace_artifact.py`, `server/services/artifact_*.py` | | | ✔ | Thay bằng watcher trong Docker sandbox |
| Code execution | `utils/code_execution.py`, `runtime/node/executor/python_executor.py`, `functions/function_calling/{code_executor,uv_related}.py` | | | ✔ | Docker sandbox bắt buộc |
| Workspace path security | `utils/workspace_paths.py`, `tests/test_*security.py` | ✔ (rule + test cases) | | ✔ | Port sang TS/Jest |
| Python SDK / CLI | `runtime/sdk.py`, `run.py`, `server/routes/execute_sync.py` | | | ✔ | TS SDK + CLI tương đương |
| Frontend builder | `frontend/src/pages/WorkflowView.vue`, `components/{WorkflowNode,WorkflowEdge,FormGenerator,DynamicFormField}.vue` | | | ✔ React Flow | Tham chiếu UX: form từ schema, lưu vuegraph layout riêng |
| Launch console | `frontend/src/pages/LaunchView.vue` | | | ✔ | Tham chiếu state machine Connecting→Waiting→Launching→Running |
| Sprites & avatar binding | `frontend/public/sprites/`, `frontend/src/utils/spriteFetcher.js` | ✔ (nếu license cho phép) | | | 12 nhân vật × 4 hướng × 3 frame → Phaser spritesheet |
| Visualizer 1.0 | `v1.1.6:visualizer/` | | | ✔ | Tham chiếu avatar theo role |
| Docs | `docs/user_guide/en/**` | ✔ (spec tham chiếu) | | | execution_logic, dynamic_execution, field_specs, mcp |
| Docker/compose | `Dockerfile`, `compose.yml`, `frontend/Dockerfile` | | ✔ (pattern multi-stage, non-root) | ✔ | Thêm sandbox image |
| MCP example server | `mcp_example/mcp_server.py` | ✔ (dùng test MCP) | | | FastMCP |
