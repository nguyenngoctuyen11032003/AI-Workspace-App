# PROVIDER_INTEGRATION.md — Provider Adapter: Claude Code, Codex, OpenHands, Generic CLI

Tác giả: Agent AI Integration — Phase 0, chỉ tài liệu (không có mã ứng dụng). Ngày: 2026-10-30. Trạng thái: DRAFT chờ phê duyệt.
Nhãn: **[Đã xác minh]** (có output thật hoặc nguồn chính thức trong `docs/research/`), **[Đề xuất]**, **[Giả định]**, **[Cần người dùng phê duyệt]**.
Nguồn chính: `DECISION_LOG.md` (D-004..D-008, D-010, D-014), `research/CLAUDE_CODE_INTEGRATION.md`, `research/CODEX_INTEGRATION.md`, `research/OPENHANDS_ANALYSIS.md`, `product/FUNCTIONAL_REQUIREMENTS.md` (FR-076..FR-100), `SECURITY.md` §4–5, `architecture/RESOURCE_MANAGEMENT.md`, `design/SCREEN_SPECIFICATION.md` §3.11/§3.17.

## 1. Mục đích

Định nghĩa lớp **Provider Adapter** duy nhất mà phần còn lại của AI WORKSPACE (Scheduler, Workflow Conductor, Approval Center, Virtual Office, Cost) dùng để nói chuyện với "bộ não" AI. Core không biết đang chạy Claude Code, Codex hay OpenHands; mọi khác biệt nằm trong adapter. Tài liệu này hợp nhất hai interface không đồng nhất trong research (`BrainAdapter` của Agent 05 và `AgentAdapter` của Agent 06) thành một interface 13 operation theo FR-081.

## 2. Phạm vi

- Trong: interface TypeScript, event chuẩn hóa, capability flags, thiết kế 4 adapter, luồng kết nối/đăng nhập, quy tắc xác thực, concurrency/rate-limit, usage/cost, lỗi/retry, cancel/resume, cầu nối approval, credential, Windows, chiến lược test, bảng đã xác minh vs giả định.
- Ngoài: thuật toán chọn agent (`AGENT_SCHEDULER.md`), policy engine chi tiết (`SECURITY.md` §5), sandbox Docker (`SECURITY.md` §2), schema DB (`DATABASE_DESIGN.md`; chỉ nêu delta cần thiết ở §24).
- Chỉ Claude Code, Codex, OpenHands, Generic CLI. API key provider (Anthropic API, OpenAI API...) là FR-098, **chưa chốt (OI-06)** nên chỉ để chỗ gắn (§18.4).

## 3. Kiến trúc tổng thể

```
Scheduler / Conductor ──► ProviderRegistry ──► ProviderAdapter (13 op) ──► ProviderEvent stream
                                │                  │  ClaudeCodeAdapter  → @anthropic-ai/claude-agent-sdk → claude(.exe)
   ApprovalBridge ◄─────────────┤                  │  CodexAdapter       → codex app-server (stdio) | codex exec --json
   UsageRecorder  ◄─────────────┤                  │  OpenHandsAdapter   → openhands-agent-server (REST + WS)
   RateLimitGuard ◄─────────────┘                  │  GenericCliAdapter  → cấu hình do Owner duyệt
        │                                          └─ MockProviderAdapter (chỉ CI, §20.5)
        ▼
 outbox → agent_events (append-only, D-014) → Redis Stream → Office / UI / Cost
```

Quy tắc bất biến **[Đề xuất]**: (1) adapter không ghi DB trực tiếp, chỉ phát `ProviderEvent`; (2) mọi thứ UI/Office thấy đi qua event đã persist (FR-118); (3) adapter không gọi ngược Scheduler; (4) adapter không chứa chính sách (policy ở ApprovalBridge); (5) mọi payload đi qua redaction (`SECURITY.md` §4.4) trước khi rời adapter.

## 4. Provider Adapter interface (13 operation, FR-081)

```ts
export type ProviderKind = 'claude_code' | 'codex' | 'openhands' | 'generic_cli' | 'mock'; // 'mock' chỉ tồn tại khi AIW_TEST_MODE=ci (§20.5)
export type OpId = 'connect'|'disconnect'|'authenticate'|'checkConnection'|'listModels'|'createSession'
  |'resumeSession'|'sendMessage'|'streamEvents'|'cancelTask'|'getUsage'|'getError'|'getSessionHistory';

export interface ProviderAdapter {
  readonly kind: ProviderKind;
  readonly capabilities: ProviderCapabilities;                         // §6, tĩnh + có thể thu hẹp theo phiên bản phát hiện
  connect(cfg: ProviderConnectConfig): Promise<ProviderHandle>;        // 1. phát hiện binary/server, KHÔNG đăng nhập hộ
  disconnect(h: ProviderHandle, o?: { force?: boolean }): Promise<void>; // 2. đóng process/socket; không logout CLI
  authenticate(h: ProviderHandle, o?: AuthOptions): Promise<AuthState>;  // 3. đọc trạng thái + trả hướng dẫn login; không nhận mật khẩu/token
  checkConnection(h: ProviderHandle): Promise<ConnectionHealth>;       // 4. version + auth + (nếu có) quota; idempotent, ≤ 10 s
  listModels(h: ProviderHandle): Promise<ModelInfo[]>;                 // 5. nguồn cho Agent Builder (FR-093)
  createSession(h: ProviderHandle, r: CreateSessionRequest): Promise<SessionRef>; // 6
  resumeSession(h: ProviderHandle, ref: SessionRef, o?: ResumeOptions): Promise<SessionRef>; // 7
  sendMessage(s: SessionRef, m: UserMessage): Promise<TurnRef>;        // 8. enqueue một turn, trả về ngay khi provider nhận
  streamEvents(s: SessionRef, o?: { afterSeq?: number }): AsyncIterable<ProviderEvent>; // 9. hot stream, seq tăng dần theo session
  cancelTask(s: SessionRef, t?: TurnRef, o?: CancelOptions): Promise<CancelResult>;      // 10
  getUsage(s: SessionRef | ProviderHandle, scope: UsageScope): Promise<UsageReport>;     // 11
  getError(s: SessionRef | ProviderHandle): Promise<ProviderErrorInfo | null>;           // 12. lỗi gần nhất đã phân loại (§15)
  getSessionHistory(s: SessionRef, p?: { cursor?: string; limit?: number }): Promise<HistoryPage>; // 13
}

export interface ProviderConnectConfig {
  providerConfigId: string; binaryPath?: string; configDir?: string;   // dir của CLI (CODEX_HOME...), KHÔNG chứa secret
  serverUrl?: string; serverKeyRef?: string;                           // OpenHands: key nằm trong OS credential store (§18)
  env?: Record<string, string>;                                        // chỉ biến whitelist (§18.2)
}
export interface CreateSessionRequest {
  agentId: string; taskId?: string; cwd: string; additionalDirs?: string[];
  model: string; effort?: 'low'|'medium'|'high'|'xhigh'|'max';
  persona: string;                                                     // append vào system prompt / developerInstructions
  permission: { mode: 'ask'|'accept_edits'|'plan'|'deny_unlisted'; allow: string[]; deny: string[] };
  budget?: { maxTurns?: number; maxUsd?: number };                     // maxUsd là proxy quota (RESOURCE §5)
  outputSchema?: object; ephemeral?: boolean; resume?: SessionRef; fork?: boolean;
  approver: (r: ApprovalRequest) => Promise<ApprovalDecision>;         // do ApprovalBridge cung cấp (§17)
}
export interface SessionRef { providerSessionId: string; providerKind: ProviderKind; resumeToken?: string; cwd: string; }
export interface TurnRef { turnId: string; providerTurnId?: string; resultIndex?: number; }
export interface UserMessage { text: string; images?: string[]; outputSchema?: object; }
export interface CancelOptions { mode: 'soft' | 'hard'; softTimeoutMs?: number; reason: 'user'|'budget'|'emergency_stop'|'timeout'|'policy'; }
export interface CancelResult { outcome: 'interrupted'|'closed'|'killed'|'not_running'; sessionResumable: boolean; turnCompleted: boolean; }
export type AuthState =
  | { state: 'authenticated'; method: 'subscription'|'api_key'|'token'|'unknown'; plan?: string; account?: { emailMasked?: string; org?: string } }
  | { state: 'login_required'; steps: LoginStep[] }                    // chỉ hướng dẫn; UI hiển thị lệnh, không tự chạy (SCREEN §3.11)
  | { state: 'unsupported_auth'; reason: string };                     // vd. CLI đang dùng API key/Bedrock trong khi chính sách yêu cầu subscription
export interface LoginStep { order: number; title: string; command?: string; url?: string; note?: string }
export interface ConnectionHealth { status: 'healthy'|'degraded'|'rate_limited'|'auth_error'|'down'|'not_configured';
  installed: boolean; version?: string; versionOk: boolean; auth: AuthState['state']; quota?: QuotaSnapshot; checkedAt: string; detail?: string }
export interface QuotaSnapshot { windows: Array<{ name: 'five_hour'|'seven_day'|'primary'|'secondary'; usedPct: number; resetsAt?: number }>; source: 'provider'|'estimated' }
export interface ModelInfo { id: string; displayName: string; contextWindow?: number; maxOutput?: number; pricing?: { inputPerM: number; outputPerM: number }; isDefault?: boolean; source: 'provider'|'static' }
export interface UsageReport { tokens: { input: number; output: number; cacheRead?: number; cacheWrite?: number; reasoning?: number };
  costUsd?: number; costBasis: 'list'|'billed'|'estimated'|'none'; estimated: boolean; byModel?: Record<string, number>; asOf: string }
```

Operation không hỗ trợ **phải** ném `ProviderError{code:'NOT_SUPPORTED', op}` (FR-081); `capabilities` cho UI biết trước để ẩn nút. Không operation nào nhận password hoặc token (FR-084/085) **[Đã xác minh theo FR]**.

Bảng ánh xạ operation → cơ chế thật:

| Op | ClaudeCodeAdapter | CodexAdapter (app-server) | OpenHandsAdapter | GenericCliAdapter |
|---|---|---|---|---|
| connect | resolve binary SDK-bundled hoặc `AIW_CLAUDE_BIN`; `claude --version` | resolve `codex.exe`; spawn `codex app-server`; `initialize` + `initialized` | GET `/server_info`; version-gate | kiểm `command` tồn tại + `--version` (nếu cấu hình) |
| disconnect | `Query.close()` mọi session | `thread/*` đóng, kill app-server (process tree) | đóng WS; không dừng server do người dùng khởi tạo | kill process |
| authenticate | `claude auth status` (JSON) | `codex login status` (exit code) + `account/read` | OpenHands không có login subscription; kiểm LLM profile / ACP (§9.5) | lệnh `loginCheck` cấu hình |
| checkConnection | version + auth + `rate_limit_event` mới nhất | version + `account/read` + `account/rateLimits/read` | `/server_info`, `/alive` | exit code lệnh health |
| listModels | `Query.supportedModels()` + alias `haiku/sonnet/opus/fable` | `model/list` | `/api/acp/models` (ACP) hoặc option-service (native) | `NOT_SUPPORTED` → danh sách tĩnh trong config |
| createSession | `query({prompt: AsyncIterable, options})` | `thread/start` | POST `/api/conversations` | spawn process (1 process = 1 session) |
| resumeSession | `options.resume = sessionId` | `thread/resume` / `exec resume <id>` | conversation đã persist; WS `after_seq` | `NOT_SUPPORTED` (mặc định) |
| sendMessage | `input.push({type:'user'})` (streaming input) | `turn/start` | `ConversationClient.sendMessage` | ghi stdin / spawn lại |
| streamEvents | `for await (m of query)` → map | notifications + server-requests → map | `/sockets/session/{id}?after_seq=N` → map | parser cấu hình |
| cancelTask | `interrupt()` → `close()` | `turn/interrupt` | pause (`ConversationClient`) | SIGINT-tương-đương → taskkill |
| getUsage | cộng dồn từ `result` (delta) | `thread/tokenUsage/updated` | `stats` (`usage_to_metrics`) | `estimated` |
| getError | lỗi phân loại cuối của session/provider | như trên | như trên | như trên |
| getSessionHistory | nguồn chính: `agent_events` của ta; phụ: transcript do SDK/CLI | `thread/read` | `RemoteEventsList.search` | chỉ `agent_events` |

## 5. Event chuẩn hóa (`ProviderEvent`)

Envelope **[Đề xuất]** (học từ OpenHands: tách envelope/payload, durable vs transient vs delta; D-014):

```ts
export interface ProviderEvent<K extends ProviderEventKind = ProviderEventKind> {
  id: string;               // uuid v7 do adapter sinh
  seq: number;              // tăng dần theo session, bắt đầu 1; cursor resume cho streamEvents(afterSeq)
  ts: string;               // thời điểm adapter nhận (ISO); providerTs nếu có trong payload
  provider: ProviderKind; sessionId: string; turnId?: string; agentId: string; taskId?: string;
  durability: 'durable' | 'transient';   // transient (delta, progress) không persist, chỉ đẩy realtime
  kind: K; payload: PayloadOf<K>; redacted: true;      // raw chỉ giữ ở debug, không persist mặc định
}
export type ProviderEventKind =
  | 'session.started' | 'session.state_changed' | 'session.ended'
  | 'turn.started' | 'turn.completed' | 'turn.failed' | 'turn.interrupted'
  | 'message.delta' | 'message.completed'
  | 'thinking.started' | 'thinking.delta' | 'thinking.completed'
  | 'tool.started' | 'tool.output' | 'tool.finished'
  | 'file.changed' | 'subagent.started' | 'subagent.finished' | 'plan.updated'
  | 'approval.requested' | 'approval.resolved'
  | 'usage.updated' | 'rate_limit.updated' | 'provider.status_changed' | 'error' | 'heartbeat';
export type ToolKind = 'read'|'search'|'edit'|'write'|'delete'|'move'|'execute'|'fetch'|'mcp'|'think'|'subagent'|'other'; // cùng taxonomy với ACP (OpenHands)
export type SessionState = 'starting'|'idle'|'running'|'waiting_approval'|'waiting_input'|'paused'|'finished'|'error'|'stuck';
// payload tiêu biểu
// tool.started  : { toolCallId, name, kind: ToolKind, inputSummary: string(≤200, đã redact), parentToolCallId?: string|null, path?: string }
// tool.finished : { toolCallId, ok: boolean, exitCode?: number, outputSummary: string(≤500), durationMs?: number }
// approval.requested: { approvalRef, tool: string, kind: ToolKind, inputHash: string, inputRedacted: unknown, reason?: string, availableDecisions: Decision[] }
// usage.updated : UsageReport (đã là delta của lượt này + cumulative của session)
// error         : ProviderErrorInfo (§15)
```

**Bảng ánh xạ nguồn → event chuẩn** (cột trái là thực tế đã quan sát):

| Chuẩn | Claude (SDK `SDKMessage`) **[Đã xác minh]** | Codex app-server (camelCase v2) **[Đã xác minh]** | OpenHands (`SessionFrame`/Event) **[Đã xác minh]** |
|---|---|---|---|
| `session.started` | `system/init` (`session_id`, `model`, `apiKeySource`) | `thread/started` | `POST /api/conversations` ok + `ConversationStateUpdateEvent` |
| `turn.started` | message `user` đầu turn / `result_index` mới | `turn/started` | `execution_status=running` |
| `message.delta` | `stream_event` `text_delta` (cần `includePartialMessages`) | `item/agentMessage/delta` | frame `delta` kind `text` |
| `message.completed` | `assistant` block `text` | `item/completed` `agentMessage` | `MessageEvent` (durable) |
| `thinking.*` | `system/thinking_tokens`, block `thinking` | `item/reasoning/*` | `ActionEvent.thought`, delta kind `reasoning` |
| `tool.started` | `assistant` block `tool_use` (`parent_tool_use_id` = subagent) | `item/started` `commandExecution`/`fileChange`/`mcpToolCall` | `ActionEvent` (`tool_name`, `tool_call_id`) / `ACPToolCallEvent` |
| `tool.finished` | `user` block `tool_result` (`is_error`) | `item/completed` (`status`, `exitCode`) | `ObservationEvent` (`action_id`) / `UserRejectObservation` |
| `file.changed` | suy từ tool `Edit/Write` | `item/completed` `fileChange.changes[]` | `FileEditorAction` observation |
| `approval.requested` | `canUseTool` callback | server-request `item/*/requestApproval` | `execution_status=waiting_for_confirmation` |
| `usage.updated` | `result.usage`, `total_cost_usd`, `modelUsage` | `thread/tokenUsage/updated` | `stats` → `usage_to_metrics` |
| `rate_limit.updated` | `rate_limit_event.unifiedWindows` | `account/rateLimits/updated` | không có kênh riêng (lỗi LLM 429) **[Giả định]** |
| `turn.completed/failed` | `result` (`subtype`, `is_error`) | `turn/completed` `status` | `finished` / `error` |
| `turn.interrupted` | `interrupt()` ack + `result` kết thúc sạch | `turn/completed` `status:'interrupted'` | `PauseEvent` |
| `error` | `system/api_retry`, `result.api_error_status`, stderr | notification `error` (`willRetry`) | `ServerErrorEvent`, `AgentErrorEvent` |

Quy tắc: event không map được **bị bỏ qua có chủ đích** ("the envelope is allowed to grow", OpenHands) và đếm vào metric `provider_unmapped_event_total{provider,type}`; tuyệt đối không bịa event để làm đẹp UI/Office (FR-118, FR-160). Event `durable` đi outbox → `agent_events` (tên `session.started`, `tool.called` ... của DB sẽ do mapper domain chuyển từ `kind`); `transient` chỉ qua Redis Pub/Sub.

## 6. Capability flags theo provider

```ts
export interface ProviderCapabilities {
  streaming: boolean; partialText: boolean; thinkingEvents: boolean; interrupt: boolean; steer: boolean;
  resume: boolean; fork: boolean; interactiveApproval: boolean; structuredOutput: boolean;
  sessionsPerProcess: 'one' | 'many'; usageTokens: boolean; usageCostBilled: boolean; quotaQuery: boolean;
  modelList: boolean; loginFromUi: boolean; nativeHistory: boolean; builtInSandbox: 'none'|'os'|'container'; mcp: boolean; subagents: boolean;
}
```

| Flag | Claude (SDK) | Codex app-server | Codex exec (fallback) | OpenHands | Generic CLI |
|---|---|---|---|---|---|
| streaming / partialText | ✔ / ✔ | ✔ / ✔ | ✔ (item-level) / ✘ | ✔ / ✔ | tùy parser / ✘ |
| thinkingEvents | ✔ | ✔ (summary) | ✔ (`reasoning` item) | ✔ | ✘ |
| interrupt / steer | ✔ / ✘ (chưa xác minh) | ✔ / ✔ (`turn/steer`) | ✘ (kill) / ✘ | ✔ (pause) / ✔ (send khi running **[Giả định]**) | ✘ / ✘ |
| resume / fork | ✔ / ✔ | ✔ / ✔ | ✔ / ✔ (`exec fork`) | ✔ / ✔ | ✘ / ✘ |
| interactiveApproval | ✔ (`canUseTool`) | ✔ | ✘ (`never` hoặc auto-review) | ✔ (`confirmation_policy`) | ✘ |
| structuredOutput | ✔ (`json_schema`) | ✔ (`outputSchema`) | ✔ (`--output-schema`) | **[Giả định]** | ✘ |
| sessionsPerProcess | one | **many** | one | many | one |
| usageTokens / usageCostBilled | ✔ / ✘ (list price) | ✔ / ✘ | ✔ / ✘ | ✔ / ✔ nếu LLM API key; ✘ nếu ACP | ✘ (ước lượng) |
| quotaQuery | ✔ (`rate_limit_event`) | ✔ (`account/rateLimits/read`) | ✘ | ✘ | ✘ |
| modelList / loginFromUi | ✔ / ✘ (`claude auth login` tương tác ngoài app) | ✔ / ✔ (`account/login/start`) **[Cần người dùng phê duyệt]** | ✘ / ✘ | ✔ (ACP) / ✘ | ✘ / ✘ |
| builtInSandbox | none (chính sách, không phải ranh giới — D-007) | os (Windows MXC/elevated) | os | container (Docker runtime) | none |
| mcp / subagents | ✔ / ✔ (`Task`/`Agent` tool) | ✔ / ✔ (`collabAgentToolCall`) | ✔ / ✘ | ✔ / ✔ (`TaskAction`, child conversation) | ✘ / ✘ |

`capabilities` có thể bị thu hẹp lúc `connect` theo version thật (vd. app-server không có `turn/steer`). Agent Builder và Provider Hub đọc flags này để ẩn/disable tính năng (ý tưởng "capability flag điều khiển UI", clean-room từ MVO).

## 7. ClaudeCodeAdapter (D-004)

**Cơ chế [Đã xác minh]**: `@anthropic-ai/claude-agent-sdk` (đã test 0.3.314; SDK bundle binary `claude` cùng version), kế thừa OAuth login của CLI (`apiKeySource: "none"`, không cần `ANTHROPIC_API_KEY`). Raw CLI (`claude -p --output-format stream-json`) chỉ làm healthcheck/fallback (`RawCliClaudeAdapter`, cùng `mapMessage`).

Cấu hình `Options` bắt buộc cho mọi session **[Đề xuất, dựa trên CLAUDE_CODE_INTEGRATION §9.2]**:

| Option | Giá trị | Lý do |
|---|---|---|
| `cwd`, `additionalDirectories` | worktree của task (D-009), thư mục tối thiểu | cô lập |
| `model`, `effort` | tường minh từng agent | mặc định lấy từ user settings (`fable`) → tránh tốn quota ngoài ý muốn |
| `systemPrompt` | `{type:'preset', preset:'claude_code', append: persona}` | persona AI employee |
| `permissionMode` + `allowedTools`/`disallowedTools` | `default` + whitelist an toàn; `acceptEdits`/`auto` chỉ theo role | không dùng `bypassPermissions` ngoài sandbox |
| `canUseTool` | → ApprovalBridge (§17) | approval tương tác |
| `hooks.PreToolUse` | in-process, `policyGuard` (deny/rewrite theo `SECURITY.md` §3, `updatedInput`) | policy cứng |
| `settingSources: ['user']`, `strictMcpConfig: true`, `settings:{disableAllHooks:true}` | | repo lạ vẫn nạp `.claude/settings.json` + `.mcp.json` ở `-p` nếu không chặn |
| `includePartialMessages: true` | | `message.delta` |
| `maxTurns`, `maxBudgetUsd`, `abortController` | từ `budget` | hard stop |
| `env` | **thay thế toàn bộ env** → dựng từ whitelist (§18.2), không `ANTHROPIC_API_KEY`, không `CLAUDE_CODE_OAUTH_TOKEN` | secret hygiene |
| `pathToClaudeCodeExecutable` | `AIW_CLAUDE_BIN` nếu Owner muốn dùng bản CLI đã cài | mặc định dùng bản SDK bundle (deterministic) |
| `sessionId` | UUID do ta sinh | biết trước ID, gắn `agent_sessions.provider_session_id` |

Vòng đời: 1 agent = 1 process sống lâu (streaming input) trong lúc làm task; idle > N phút → `close()` rồi `resume` khi có task mới (giải phóng ~230 MB RSS/process, RESOURCE §3). Startup ~3–5 s/process **[Đã xác minh]** → giữ process sống thay vì spawn mỗi turn.

Điểm cần biết **[Đã xác minh]**: `total_cost_usd` là **tổng tích lũy** của session khi `--resume` → adapter phải lưu `lastCumulativeCost` và phát delta; `costBasis:"list"` ⇒ không phải tiền thật với subscription; `--bare` không đọc OAuth → **cấm dùng**; `-p` khởi động Manual mode ⇒ mọi tool chưa allow bị deny nếu không có host (đã có `canUseTool`).

## 8. CodexAdapter (D-005)

**Chính: `codex app-server` stdio JSON-RPC** (1 process, nhiều thread = nhiều AI employee). **Dự phòng: `codex exec --json`** (SDK `@openai/codex-sdk` chỉ làm implementation của fallback/batch; không approval callback, không interrupt mềm). **Phụ: rollout watcher** (`~/.codex/sessions/**/*.jsonl`) chỉ để quan sát session ngoài app, không làm nguồn chính (3 format khác nhau: snake_case exec / camelCase app-server / PascalCase rollout) **[Đã xác minh]**.

Chuỗi gọi **[Đã xác minh từ docs + bindings 0.164, chưa chạy live vì CLI chưa cài]**:

```jsonl
{"method":"initialize","id":0,"params":{"clientInfo":{"name":"ai_workspace","title":"AI Workspace","version":"x"},"capabilities":{"experimentalApi":true}}}
{"method":"initialized","params":{}}
{"method":"thread/start","id":10,"params":{"cwd":"<worktree>","model":"<m>","approvalPolicy":"on-request","sandbox":"workspace-write","developerInstructions":"<persona>","serviceName":"ai_workspace"}}
{"method":"turn/start","id":30,"params":{"threadId":"<id>","input":[{"type":"text","text":"..."}],"effort":"medium"}}
{"method":"turn/interrupt","id":31,"params":{"threadId":"<id>","turnId":"<id>"}}
```

Quy tắc thiết kế **[Đề xuất]**:
- Framing: mỗi dòng 1 JSON, **không có field `jsonrpc`**; phân loại theo `id`+`result|error` (response), `id`+`method` (server-request, phải trả lời), `method` (notification).
- `-32001` (server overloaded) → retry backoff+jitter; `-32602` → lỗi `INVALID_REQUEST`, không retry.
- Cấu hình **do app sinh, không kế thừa** `~/.codex/config.toml` của user (SECURITY §2.5): per-thread `config` + `sandbox:'workspace-write'`, `approvalPolicy:'on-request'`, `network_access=false`, `[windows] sandbox="mxc"` nếu build ≥ 26100.9278 (máy dev 26300 đủ điều kiện **[Đã xác minh OS build; MXC chưa chạy thử]**), fallback `elevated`. Cấm `--yolo`/`danger-full-access`; `--full-auto` đã bị gỡ; `approval_policy="untrusted"` đã retire; chỉ dùng `on-request|never|granular` **[Đã xác minh trên 0.164]**.
- Supervisor: app-server chết → restart → `thread/resume` các thread đang active → `thread/list` đối chiếu; sau 3 lần chết liên tiếp trong 5 phút chuyển `degraded` và dùng `exec` fallback cho task read-only.
- Binary: `npm i -g @openai/codex` rồi resolve `codex.exe` thật; **không** dùng binary bundled trong VS Code extension (đổi theo update). Sau khi cài: chạy `codex app-server generate-ts --out packages/codex-protocol/generated` và commit theo version; CI so hash bindings mỗi đêm (canary drift, §20.6).
- `ItemCompleted` là trạng thái authoritative; delta chỉ để hiển thị. Mọi message từ Codex là **untrusted data** (không eval, escape khi render).
- Đọc `thread/tokenUsage/updated` cho usage; **không có cost** → ước lượng bằng `provider_configs.pricing` và gắn `estimated:true`.

## 9. OpenHandsAdapter (D-006)

### 9.1 Bản chất **[Đã xác minh]**
Repo `OpenHands/OpenHands` hiện là **Agent Canvas** (UI); agent/runtime là **Agent Server** Python (`openhands-agent-server`, repo `software-agent-sdk`, MIT) chạy như dịch vụ ngoài. Adapter chỉ nói REST/WS với nó qua `@openhands/typescript-client` (pin 1.59.0). **Không vendor mã nguồn, không nhúng Python vào backend**. Không dùng tên/logo OpenHands (trademark).

### 9.2 Khởi chạy server **[Đã xác minh lệnh; chưa chạy trên máy này]**
```
uvx --python ">=3.12,<3.15" --from openhands-agent-server==1.59.0 --with openhands-sdk==1.59.0 --with openhands-tools==1.59.0 --with openhands-workspace==1.59.0 agent-server
```
hoặc image `ghcr.io/openhands/agent-server:<ver>-python`. Server bind loopback; auth bằng header `X-Session-API-Key` (64 ký tự, env `OH_SESSION_API_KEYS_0`), secret mã hóa bằng `OH_SECRET_KEY` (thiếu → 503 khi tạo conversation). Lần chạy `uvx` đầu có thể chờ tới 10 phút (tải Python+package) → `connect` poll `/server_info` tới 200/401, timeout 600 s, hiển thị tiến trình.

### 9.3 Mapping **[Đề xuất]**
| Operation | Cách làm |
|---|---|
| createSession | `POST /api/conversations {agent|agent_profile_id, workspace, confirmation_policy, security_analyzer, secrets, tags{task_id,role,agent_id}, max_iterations, stuck_detection:true}`; `workspace.kind = LocalWorkspace` (Option 1, **không sandbox**, cảnh báo rõ) hoặc `DockerExecutionWorkspace` (Tier 2, D-007; Windows native **không** hỗ trợ container-per-conversation, cần WSL2) |
| streamEvents | WS `/sockets/session/{id}?after_seq=N`; `durable` → persist với `seq`; `transient`/`delta` → realtime; reconnect backoff 1→30 s + jitter 30%; frame lạ bị bỏ qua |
| approval | `confirmation_policy: ConfirmRisky{threshold:'HIGH', confirm_unknown:true}` (Tier 2) / `AlwaysConfirm` (mặc định an toàn); `waiting_for_confirmation` → `approval.requested`; trả lời bằng `respondToConfirmation({accept, reason})` |
| cancelTask | pause; dừng hẳn = `deleteConversation` hoặc để idle; stuck detection phát `stuck` → `session.state_changed` |
| usage | `ConversationStateUpdateEvent{key:'stats'}` → `usage_to_metrics[usage_id].accumulated_cost` (cumulative theo `usage_id`: `default`, `condenser`, `profile:*`) |
| exec/terminal | `RemoteWorkspace.executeCommand` (bash-as-a-service) — chỉ cho terminal tab, vẫn qua policy |

### 9.4 Version gate **[Đề xuất]**
Contract đổi nhanh (Canvas pin `minimumAgentServer` 1.57.0, nhiều field mới mỗi release). `connect` đọc `/server_info` (version, `execution_runtime`) và feature-probe; version < tối thiểu → `VERSION_UNSUPPORTED`; field/endpoint 404 → hạ capability thay vì crash.

### 9.5 Điểm xung đột cần quyết **[Cần người dùng phê duyệt]**
1. **Credential**: agent native cần LLM API key (đụng FR-098/OI-06). ACP sub-mode (Claude Code/Codex chạy bên trong OpenHands, tận dụng login CLI) thì OpenHands "materialize" `CODEX_AUTH_JSON` → `~/.codex/auth.json` trong container, **mâu thuẫn D-008** (app không đọc/copy `auth.json`). Đề xuất: **MVP chỉ hỗ trợ OpenHands native + LLM key trong OS keychain; ACP sub-mode tắt** cho đến khi Owner duyệt.
2. **LookupSecret**: server tự gọi URL của ta để lấy secret lúc spawn. Đề xuất endpoint loopback, token một lần, TTL 30 s, chỉ trả cho conversation vừa tạo; ngoài ra không có endpoint nào trả giá trị secret (SECURITY §4.1.5).
3. FR-078 ghi "cơ chế chưa rõ" — nay đã rõ (mục này); cần cập nhật FR.

## 10. GenericCliAdapter (FR-079, ưu tiên S/P2)

Mục tiêu: thêm CLI mới không sửa core (NFR-101). Cấu hình khai báo, validate bằng zod, lưu trong `provider_configs.settings` **[Đề xuất]**:

```ts
interface GenericCliManifest {
  id: string; displayName: string; command: string; versionArgs: string[];          // vd ["--version"]
  loginCheck?: { args: string[]; okExitCodes: number[]; notLoggedInPattern?: string };
  loginSteps: LoginStep[];                                                          // hướng dẫn, không tự chạy
  invoke: { args: string[]; promptVia: 'stdin' | 'arg'; cwdFlag?: string; resumeArgs?: string[] };
  output: { format: 'jsonl' | 'text'; map?: Array<{ match: JsonPathPredicate; emit: ProviderEventKind; fields: Record<string, JsonPath> }> };
  env: string[];                 // tên biến cho phép chuyển tiếp (whitelist), mặc định []
  capabilities: Partial<ProviderCapabilities>;   // mặc định tất cả false
  usage: { source: 'none' | 'jsonpath'; tokens?: JsonPath };                      // không có → ước lượng, estimated:true (FR-096)
}
```
Quy tắc: (1) mọi thứ không khai báo = `NOT_SUPPORTED`; (2) `spawn(command, args, {shell:false})`, không ghép chuỗi shell, prompt qua stdin khi dài (giới hạn argv Windows); (3) parser **khai báo (JSONPath), không `eval`/hàm do người dùng viết**; (4) manifest tạo/sửa là hành động Owner-only và cần approval loại `permission` vì thực thi lệnh tùy ý; (5) vẫn đi qua denylist/ policy engine và env whitelist; không có credential riêng, không đọc file auth của CLI lạ; (6) text-only CLI chỉ phát `message.completed` + `turn.completed` (không `tool.*`) — Office không có gì để animate ngoài Working/Idle, đúng sự thật.

## 11. Luồng kết nối từng provider (FR-082/083)

Quy trình chung: `connect` (phát hiện) → `authenticate` (đọc trạng thái) → nếu thiếu: UI hiển thị **hướng dẫn từng bước kèm lệnh, không chạy tự động** (SCREEN §3.11) → người dùng làm ngoài app → nút "Kiểm tra lại" gọi `checkConnection`. Trạng thái: `not_configured → installed_not_logged_in → healthy | degraded | rate_limited | auth_error | down`.

### 11.1 Claude Code
| Bước | Lệnh | Kết quả mong đợi |
|---|---|---|
| Cài | SDK đã bundle binary (`@anthropic-ai/claude-agent-sdk-win32-x64/claude.exe`) → thường không cần cài riêng **[Đã xác minh]**. CLI độc lập đặt tại `C:\Users\<u>\.local\bin\claude` trên máy dev **[Đã xác minh]**; lệnh cài CLI chính thức không có trong research → hiển thị link docs, không tự suy ra **[Giả định]** | — |
| Version | `claude --version` | `2.1.312` trên máy dev |
| Auth | `claude auth status` (JSON mặc định) | `loggedIn:true, authMethod:"claude.ai", apiProvider:"firstParty", subscriptionType:"team"` |
| Login | `claude auth login` (tương tác, mở trình duyệt, **chạy ngoài app**) | rồi "Kiểm tra lại" |
| Logout | `claude auth logout` (người dùng tự chạy; app chỉ ngắt kết nối) | `loggedIn:false` |
| Smoke | `claude -p "Reply with exactly: PONG" --output-format json --max-turns 1 --model haiku` | `result:"PONG"`, `is_error:false` |

### 11.2 Codex
| Bước | Lệnh | Kết quả |
|---|---|---|
| Phát hiện | `Get-Command codex` / `where codex` | trên máy dev: **không có** → `installed:false` **[Đã xác minh]** |
| Cài | `npm install -g @openai/codex` rồi `codex --version` | latest 0.166.0 (npm); bundled extension 0.164.0-alpha.11.1 |
| Auth | `codex login status` | `Logged in using ChatGPT` exit 0 (đã xác minh); chưa login: `Not logged in`, exit ≠ 0 **[Giả định — chưa đo]** |
| Login | `codex login` (OAuth browser, callback `http://localhost:1455`) hoặc `codex login --device-auth` (headless, phải bật trong ChatGPT security settings) | CLI + VS Code extension **dùng chung** `~/.codex/auth.json` nên máy dev không cần login lại |
| Login từ UI | app-server `account/login/start {type:'chatgpt'}` → `authUrl` → `account/login/completed` | **[Cần người dùng phê duyệt]** vì SCREEN §3.11 quy định chỉ hiển thị lệnh |
| Không dùng | `codex login --with-api-key`, `--with-access-token` (nhập key qua stdin) | thuộc FR-098 (OI-06), chưa chốt |
| Logout | `codex logout` (người dùng tự chạy; **ảnh hưởng cả IDE/Desktop** vì chung cache) | cảnh báo trước khi hướng dẫn |

### 11.3 OpenHands
| Bước | Lệnh/kiểm tra | Ghi chú |
|---|---|---|
| Phát hiện `uv` | `uv --version` | thiếu → hướng dẫn cài uv; Python ≥3.12 do `uvx --python` lo |
| Khởi chạy | lệnh `uvx ...agent-server` (§9.2) hoặc `docker run ghcr.io/openhands/agent-server:<ver>-python` | Docker daemon **chưa chạy** lúc khảo sát (DECISION_LOG mâu thuẫn #3) |
| Health | GET `/alive`, GET `/server_info` | 200/401 = server lên; 502 từ ingress = upstream chưa sẵn |
| Auth | header `X-Session-API-Key` đọc từ OS credential store | không phải login người dùng |
| LLM | LLM profile có key trong keychain (native) | §9.5 |

## 12. Quy tắc phát hiện xác thực (auth detection)

1. **Chỉ dùng lệnh/API của chính provider**, không bao giờ đọc `~/.claude/.credentials.json`, `~/.claude.json`, `~/.codex/auth.json` (D-008, FR-085, SECURITY §4.1).
2. Claude: `isSubscription = loggedIn && authMethod==='claude.ai' && apiProvider==='firstParty'` **[Đã xác minh]**. `authMethod` khác (console/apiKey) hoặc `apiProvider` ≠ firstParty → `unsupported_auth` (cảnh báo "đang dùng API/Bedrock, chi phí thật", Owner quyết). Bổ sung kiểm chứng ở runtime: `system/init.apiKeySource === 'none'` và `Query.accountInfo()`.
3. Codex: exit code `codex login status` + chuỗi `Logged in using ChatGPT|API key`; bổ sung `account/read` (`type`, `planType`). `API key` mode → `unsupported_auth` trừ khi FR-098 được duyệt.
4. Phát hiện **hết hạn giữa chừng**: `error` map `AUTH_EXPIRED` (Claude `api_error_status` 401/403; Codex notification `error`/`account/updated`; OpenHands `STATUS$ERROR_LLM_AUTHENTICATION`) → provider `auth_error`, mọi session chạy trên pool dừng nhận task mới, thông báo người dùng (FR-089). Token tự refresh do CLI; adapter **không** xử lý refresh.
5. Thứ tự kiểm khi boot: binary → version-gate → auth → quota. Fail-fast từng bước, trạng thái cụ thể (FR-083): `NOT_INSTALLED`, `VERSION_UNSUPPORTED`, `NOT_AUTHENTICATED`.
6. Cache kết quả `checkConnection` 30 s; health timeline 24 h cho Provider Hub; thay đổi trạng thái phát `provider.status_changed` ≤ 2 s (SCREEN AC).

## 13. Concurrency và rate-limit theo provider

Thứ tự kiểm slot theo `RESOURCE_MANAGEMENT.md` §4.1: global (mặc định **5**, tối đa 18 — D-010) → pool provider → project → agent → host guard. Giá trị mặc định pool **[Đề xuất]**:

| Provider | Pool mặc định | Căn cứ | Phản ứng rate-limit |
|---|---|---|---|
| Claude Code (subscription) | 5 | 5 process song song cùng login không lỗi **[Đã xác minh]**; giới hạn thật là quota 5 giờ + tuần dùng chung | đọc `rate_limit_event`; `utilization ≥ 0.75` cảnh báo; `seven_day > 0.9` hoặc `status != allowed` → pause queue, chỉ cho lead; retry sau `resetsAt` |
| Codex (ChatGPT plan) | 3 (**[Giả định]**; cộng đồng gợi ý 2 cho Plus) | quota chung CLI/IDE/cloud, trần cứng khi đạt | `account/rateLimits/*` `usedPercent > 80` → throttle; `-32001` → backoff+jitter; hết quota = **không retry tới reset** |
| OpenHands | theo LLM provider (mặc định 2) + giới hạn container (`OH_CONVERSATION_CONTAINER_MEMORY/CPUS/PIDS_LIMIT`) | tài nguyên host | lỗi 429 từ LLM qua `ServerErrorEvent` |
| Generic CLI | 1 | không có thông tin | theo exit code/stderr pattern cấu hình |

Quy tắc chung: (1) Một `RateLimitGuard` giữ `QuotaSnapshot` mới nhất/provider (nguồn `rate_limit.updated`); (2) `RATE_LIMITED(until)`: pool vào trạng thái, task về QUEUED(reason `PROVIDER_RATE_LIMITED`), **không tự chuyển provider** (RESOURCE §9; auto-switch mặc định tắt, **[Cần người dùng phê duyệt]**); (3) tăng concurrency chỉ làm cạn quota nhanh hơn, không tăng tổng công việc; (4) cách ly: Claude dùng chung một `~/.claude` → race file giữa nhiều process chưa quan sát thấy lỗi trong 5 process, **cần đo ở 10/18** **[Giả định]**; (5) OpenHands ACP cùng provider trong một container chia sẻ HOME (race auth/lock) — lý do phụ để tắt ACP ở MVP.
Lưu ý xung đột: `DATABASE_DESIGN.md` `provider_configs.max_concurrent` mặc định **2**, khác các số trên; xem §25.

## 14. Usage / cost capture (FR-096)

| Provider | Nguồn | Cost | Đánh dấu |
|---|---|---|---|
| Claude | `result.usage`, `modelUsage`, `total_cost_usd`, `duration_ms`, `num_turns` | **ước tính theo giá niêm yết** (`costBasis:"list"`), không phải tiền trừ vào subscription; **cumulative theo session** → adapter tính delta | `costBasis:'list'`, `estimated:false` về token, UI ghi "giá niêm yết API" |
| Codex | `thread/tokenUsage/updated {total,last,modelContextWindow}`; `turn.completed.usage` (exec) | **không có** → tính từ `provider_configs.pricing` | `costBasis:'estimated'`, `estimated:true` |
| OpenHands | `stats.usage_to_metrics[*].accumulated_cost`, `token_usages[]` | thật nếu LLM API key; 0/không có nếu ACP | `billed` hoặc `none` |
| Generic | không có → ước lượng độ dài ký tự/4 | | `estimated:true` |

Ghi vào `usage_records` theo `agent_sessions`; mọi số từ quota snapshot ghi vào `rate_limit.updated`. Hard-stop budget ở ranh giới event: kiểm `usage.updated` mỗi event, vượt `budget.maxUsd` → `cancelTask(reason:'budget')` ≤ 5 s (RESOURCE AC). Cache-read token tách riêng (spawn tax ~48k token/process **[Đã xác minh]**). Provider Hub "Usage" và Cost Management phải cùng một nguồn (SCREEN AC).

## 15. Error taxonomy và retry policy

```ts
export type ProviderErrorCode =
 | 'NOT_INSTALLED'|'VERSION_UNSUPPORTED'|'NOT_AUTHENTICATED'|'AUTH_EXPIRED'|'UNSUPPORTED_AUTH'
 | 'RATE_LIMITED'|'QUOTA_EXHAUSTED'|'OVERLOADED'|'NETWORK'|'TIMEOUT'
 | 'SPAWN_FAILED'|'PROCESS_CRASHED'|'PROTOCOL_ERROR'|'SANDBOX_ERROR'
 | 'APPROVAL_DENIED'|'APPROVAL_TIMEOUT'|'BUDGET_EXCEEDED'|'CANCELLED'
 | 'SESSION_NOT_FOUND'|'INVALID_REQUEST'|'NOT_SUPPORTED'|'UNKNOWN';
export interface ProviderErrorInfo { code: ProviderErrorCode; message: string; retryable: boolean; retryAfterMs?: number; resetsAt?: number; provider: ProviderKind; op?: OpId; sessionId?: string; cause?: string /* đã redact */ }
```

| Code | Nhận diện | Retry | Chính sách |
|---|---|---|---|
| `NOT_INSTALLED`, `VERSION_UNSUPPORTED`, `NOT_AUTHENTICATED`, `UNSUPPORTED_AUTH` | connect/authenticate | không | trạng thái provider + hướng dẫn |
| `AUTH_EXPIRED` | 401/403, `account/updated` | không tự động | pool → `auth_error`, thông báo, chờ người dùng đăng nhập lại |
| `RATE_LIMITED` | Claude `api_error_status 429/529`, `status!=allowed`; Codex `-32001`/`willRetry` | có, **sau reset** nếu biết `resetsAt`; không thì 15 phút, ×2 mỗi lần liên tiếp, trần 5 giờ (RESOURCE §8.3) | CLI tự retry nội bộ (`system/api_retry`) là thông tin, không tính retry của ta |
| `QUOTA_EXHAUSTED` | "usage limit reached", Codex hard ceiling | không tới `resetsAt` | queue, không burn retry |
| `OVERLOADED`, `NETWORK`, `TIMEOUT` | 529/ECONN/spawn_timeout 60 s/silence 20 min | có, tối đa 3, backoff 2→4→8 s + jitter ±30% | rồi `FAILED` → handover (FR-111) |
| `SPAWN_FAILED`, `PROCESS_CRASHED` | exit ≠ 0 không do cancel, app-server chết | có 1–3 lần, resume cùng `sessionId` | `CLAUDE_CODE_RESUME_INTERRUPTED_TURN=1` nếu muốn tiếp turn dang dở **[Đã xác minh biến]**; app-server: `thread/resume` |
| `PROTOCOL_ERROR` | JSON hỏng, field lạ bắt buộc | không (1 lần reconnect) | bỏ frame, tăng metric; >N lỗi/phút → provider `degraded` + chuyển exec fallback (Codex) |
| `SANDBOX_ERROR` | `windowsSandbox/setupCompleted` lỗi, MXC không có | không | hướng dẫn `codex sandbox setup --elevated`; không tự hạ xuống `danger-full-access` |
| `APPROVAL_DENIED/TIMEOUT` | quyết định người/hết hạn | không | task `BLOCKED`/`FAILED` theo policy, không retry tự động cùng lệnh |
| `BUDGET_EXCEEDED` | vượt `maxUsd/maxTurns` | không | `FAILED`, owner nâng budget |

Mapping sang `FailureClass` của RESOURCE §8.3: `RATE_LIMITED|QUOTA_EXHAUSTED→RATE_LIMIT`, `AUTH_*|NOT_AUTHENTICATED→AUTH`, `PROCESS_CRASHED|SPAWN_FAILED→CRASH`, còn lại `UNKNOWN`. Retry luôn có `idempotencyKey = taskId+attempt`; tuyệt đối không retry tự động tool đã `tool.started` mà chưa có `tool.finished` (có thể đã có tác dụng phụ) — để task handover/review xử lý **[Đề xuất]**.

## 16. Cancel và resume

| | Claude | Codex app-server | OpenHands | Ghi chú |
|---|---|---|---|---|
| Soft cancel | `Query.interrupt()` (turn kết thúc sạch, giữ session) | `turn/interrupt` → `turn/completed status:interrupted` | pause | **[Đã xác minh cho Claude/Codex theo docs]** |
| Đợi soft | 5 s | 5 s | 5 s | |
| Close | `close()`: đóng stdin → grace 3 s → abort → SIGTERM (exit 143, **turn dang dở không ghi result**) | huỷ thread khỏi bộ nhớ, giữ app-server | để conversation idle | |
| Hard | `taskkill /PID <pid> /T /F` tại 10 s (`graceful_stop_timeout`) | như vậy cho process `exec` | dừng container (Tier 2) | trên Windows `process.kill` = TerminateProcess, không graceful (RESOURCE §6) |
| Resume | `options.resume = sessionId` (giữ ID; cumulative cost); `forkSession` tạo nhánh | `thread/resume` (+`excludeTurns`), `exec resume <id>`; `thread/fork` | tiếp conversation, WS `after_seq` | Transcript Claude theo slug của cwd → resume được từ cwd khác theo ID **[Đã xác minh]** |

Ngữ nghĩa **[Đề xuất]**: `cancelTask` idempotent; trả `CancelResult.sessionResumable`; nút "Emergency Stop" = hard cho toàn bộ, đặt `global.paused=true`; cancel luôn phát `turn.interrupted` hoặc `session.ended` có `reason` để Office/Activity có event thật. Không kill -9 trực tiếp trừ bước hard (để `SessionEnd` hook và cây process Bash được dọn). Resume sau restart: `agent_sessions.status` ∈ `suspended|resumable`; `provider_session_id` + `resume_token` (opaque, mã hóa) lưu DB.

## 17. Cầu nối approval → Approval Center

```ts
interface ApprovalRequest { approvalRef: string /*id JSON-RPC | toolUseID | action_id*/; sessionId: string; agentId: string; taskId?: string;
  kind: 'command'|'file_change'|'permissions'|'user_input'|'tool'; toolName: string; inputRedacted: unknown; inputHash: string; /* SHA-256 chuẩn hóa, chống TOCTOU */
  cwd?: string; reason?: string; network?: { host: string; protocol: string }; availableDecisions: Array<'accept'|'accept_for_session'|'decline'|'cancel'>; signal: AbortSignal }
interface ApprovalDecision { decision: 'accept'|'accept_for_session'|'decline'|'cancel'; updatedInput?: Record<string, unknown>; reason?: string; scope?: 'once'|'task'|'session' }
```

Luồng: adapter nhận yêu cầu → phát `approval.requested` → `ApprovalBridge` chạy **policy engine trước** (SECURITY §5.2: deny/rewrite/auto-allow theo rule) → nếu `ask`, tạo bản ghi `approvals` (loại `permission` ở Approval Center, FR/SCREEN §3.17) → người dùng Approve/Deny/Edit&Approve → `approver()` resolve → adapter trả về provider → `approval.resolved`.

| Provider | Nhận | Trả |
|---|---|---|
| Claude | `canUseTool(tool, input, {signal, toolUseID})` | `{behavior:'allow', updatedInput}` / `{behavior:'deny', message}`; `signal` abort → deny |
| Codex | server-request `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval`, `item/tool/requestUserInput` (+`serverRequest/resolved` khi turn kết thúc) | `{id, result:{decision}}` với `accept|acceptForSession|decline|cancel`; permissions: `{permissions, scope:'turn'|'session'}` |
| OpenHands | `waiting_for_confirmation` + `ActionEvent.security_risk` | `respondToConfirmation({accept, reason})` → `UserRejectObservation` |

Quy tắc **[Đề xuất]**:
- Phân tầng Claude: `PreToolUse` in-process hook = **policy cứng** (deny/rewrite, `updatedInput`); `canUseTool` = **hỏi người**. Việc `SECURITY.md` §5.1 gọi PreToolUse là "nguồn chính" và research gọi `canUseTool` là "chính" chỉ là hai tầng của cùng một engine (xem §25).
- Timeout approval mức tool: mặc định 10 phút → `decline` + `APPROVAL_TIMEOUT`, task `BLOCKED`; (SECURITY dùng 540 s cho hook HTTP, research dùng 30 phút cho `canUseTool`, RESOURCE dùng 24 h cho approval cấp workflow; hợp nhất: tool-level 10 phút, workflow-level không giới hạn mặc định theo FR-106).
- Trong lúc chờ, process/turn vẫn sống và **chiếm slot** (khác với RESOURCE §4.1 giả định `WAITING_APPROVAL` không chiếm slot). Nếu muốn nhả slot: `interrupt` + `close` rồi resume sau khi duyệt — **[Cần người dùng phê duyệt]**.
- `inputHash` được so lại ngay trước khi trả `allow` (chống đổi lệnh sau khi duyệt). "accept for session" không bao giờ ghi vào `permissions.allow`/`config.toml` của user.
- Mọi quyết định (kể cả auto) vào audit; UI chỉ hiện input đã redact (nút "show raw" theo SECURITY §5.3).
- Output của agent là dữ liệu không tin cậy; không để provider tự duyệt chính nó (`approvalsReviewer:'auto_review'` của Codex chỉ là advisory, không thay thế engine của ta).

## 18. Quy tắc credential

1. **CLI sở hữu credential.** App không đọc/copy/parse/backup `.credentials.json`, `~/.claude.json`, `auth.json`; không yêu cầu mật khẩu (FR-084/085, D-008).
2. **Env whitelist khi spawn** (SDK `env` và Codex `env` đều *thay thế* env): `PATH, SYSTEMROOT, USERPROFILE, HOME, TEMP, TMP` + `CODEX_HOME`/`CLAUDE_CONFIG_DIR` nếu cấu hình + cờ app (`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`, `DISABLE_AUTOUPDATER=1`) + `AIW_HOOK_TOKEN` khi dùng hook HTTP. `APPDATA`/`LOCALAPPDATA` có thể cần cho OAuth/sandbox trên Windows (research Claude liệt kê, SECURITY không) → **kiểm thử cả hai, mặc định theo kết quả test** **[Giả định]**. Không forward env NestJS (`DATABASE_URL`, `AIW_API_TOKEN`).
3. **Cấm đặt** `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, `CODEX_API_KEY` trong môi trường agent Tier 1; không đặt `ANTHROPIC_BASE_URL` cùng OAuth token (phá bearer auth, OpenHands docs).
4. Secret của app (OpenHands session key, LLM key nếu được duyệt, token khác): **Windows Credential Manager** qua `@napi-rs/keyring`, service `aiw`; DB chỉ có `credential_ref` (không cột value); inject qua stdin/file tạm ACL-only-user xoá sau 5 s, không qua argv (SECURITY §4.1).
5. Frontend không bao giờ nhận secret: chỉ `{id, name, maskedPreview}`; không có GET-value endpoint (FR-087).
6. Redaction trước log/DB/WS/audit (FR-090, SECURITY §4.4), kể cả `stderr` của CLI và `transcript_path`; cắt `tool_result` quá ngưỡng KB.
7. Không chia sẻ credential giữa agent (FR-091); mỗi agent chỉ đi qua adapter. Dùng nhiều `CODEX_HOME` riêng cho tách tài khoản là tùy chọn Owner, mặc định dùng chung `~/.codex`.
8. Hook HTTP (nếu dùng cho session ngoài app): bind `127.0.0.1`, Bearer token per-session; adapter SDK dùng hook in-process nên **không sửa** `~/.claude/settings.json` của user.
9. API key fallback (FR-098) = **ngoài MVP, chưa chốt**; nếu duyệt, bắt buộc keychain + adapter capability `usageCostBilled:true`.

## 19. Windows specifics

- Spawn: `child_process.spawn(file, args, {shell:false, windowsHide:true})`; trỏ thẳng tới `claude.exe`/`codex.exe`, **không** qua shim `.cmd` (npm shim cần shell; Node mới chặn spawn `.cmd` không shell — **[Giả định]** cần kiểm phiên bản Node 24 trên máy dev). Cần `npm.cmd`/`npx.cmd` thì qua `cmd.exe /c` với tham số cố định, không nối chuỗi người dùng.
- Prompt dài qua **stdin** (stream-json / `exec -`), tránh `ENAMETOOLONG`/giới hạn argv **[Đã xác minh cảnh báo trong research Claude]**.
- Path: `\\` trong JSON; transcript Claude slug = cwd với `:`/`\`/`/` → `-`, drive-letter khác hoa/thường → so khớp không phân biệt hoa thường; `fs.watch` không tin cậy → polling.
- Kill: `taskkill /PID <pid> /T /F`; `process.kill` = TerminateProcess; Job Object (`KILL_ON_JOB_CLOSE`) là nâng cao, chưa kiểm **[Giả định]**; theo dõi pid tree 5 s/lần (RESOURCE §6).
- Codex: native Windows (không bắt buộc WSL), sandbox `mxc|elevated|unelevated`; `unelevated` coi như không có sandbox (network isolation yếu); `.git/.codex/.agents` luôn read-only trong `workspace-write`; log sandbox `CODEX_HOME/.sandbox/sandbox.log`. Windows 11 build 26300 ≥ 26100.9278 nên đủ điều kiện MXC **[Đã xác minh build]**.
- OpenHands: native chạy được với `PYTHONUTF8=1`; container-per-conversation cần WSL2 (POSIX uid/gid); Docker Desktop daemon chưa chạy lúc khảo sát.
- Cấp tài nguyên: ~230 MB RSS/process Claude, startup 3–5 s **[Đã xác minh]**; RAM trống máy dev chỉ 6.4 GB → host guard `min_free_ram_mb` (RESOURCE §4.1).
- Khi app mở từ GUI, PATH tối thiểu: đừng tin PATH, resolve đường dẫn tuyệt đối lúc `connect` và lưu `binaryPath` (bài học Electron của OpenHands).

## 20. Chiến lược test

1. **Unit (parser/mapper)**: fixture NDJSON/JSON-RPC **ghi từ CLI thật** và đã redact (các output trong research §3 là mẫu đầu tiên); kiểm `mapMessage`, delta cumulative-cost, chunk boundary (2 KB overlap cho redaction), dòng JSON dở dang, `\u0073k-` bypass.
2. **Contract test chung**: một bộ test chạy với mọi adapter qua fixture: cùng kịch bản (text, tool read, tool edit cần approval, interrupt, rate-limit, crash) → chuỗi `ProviderEvent` đúng thứ tự, `seq` liên tục, `tool.started` luôn có `tool.finished` hoặc `turn.interrupted`.
3. **Live smoke (opt-in, gắn thẻ `@live`, chạy local/Windows self-hosted)**: Claude `PONG` (haiku, `--max-turns 1`), resume giữ `session_id`, `canUseTool` deny, 5 session song song (đã làm tay, tự động hóa); Codex theo checklist `CODEX_INTEGRATION.md` §11.4 khi CLI được cài. Không chạy trên CI công khai (tiêu quota, cần login).
4. **Fault injection**: 429/529, hết quota, kill process giữa turn, app-server chết, WS đứt + `after_seq`, version lạ, framing hỏng.
5. **Mock adapter chỉ cho CI** — `MockProviderAdapter` nằm trong package riêng `packages/provider-testkit`:
   - Không bao giờ nằm trong luồng sản phẩm: (a) `ProviderKind 'mock'` **không có trong enum DB** `provider_kind`, không thể lưu cấu hình; (b) `ProviderRegistry.register` ném lỗi nếu `kind==='mock'` và `process.env.AIW_TEST_MODE !== 'ci'`; (c) package testkit chỉ ở `devDependencies`, ESLint `no-restricted-imports` cấm import từ `apps/*/src` (trừ thư mục test); (d) CI bước "bundle scan": build production, grep chuỗi `MockProviderAdapter` và `provider-testkit` phải trả 0; (e) UI không có đường tạo provider mock; (f) khi chạy, mọi event mock mang `provider:'mock'` và Provider Hub từ chối hiển thị như provider thật.
   - Phát lại kịch bản từ fixture ghi thật hoặc kịch bản tay (deterministic, đồng hồ ảo). Dùng cho: test Scheduler/RateLimit (RESOURCE AC "mock trả 429"), Approval Center E2E, Virtual Office replay (`VIRTUAL_OFFICE.md` §17).
   - **Không** dùng mock để "demo" hay lấp trạng thái rỗng trong sản phẩm (FR-118/FR-160).
6. **Canary chống trôi phiên bản**: job đêm so hash `claude-agent-sdk` `sdk.d.ts` và bindings `codex app-server generate-ts`; đổi → mở issue, không tự nâng version. Pin: SDK `^0.3.314`, Codex theo bindings, agent-server `==1.59.0`.
7. **Bảo mật**: test redaction (SECURITY §9), test env không rò secret NestJS vào child, test không có đường đọc `auth.json`/`.credentials.json` (grep + sandbox test), test `inputHash` TOCTOU.

## 21. Đã xác minh vs giả định

| Mục | Trạng thái |
|---|---|
| SDK kế thừa OAuth, `apiKeySource:"none"`; `canUseTool` chặn được Write; `claude auth status` JSON; resume giữ `session_id`; `total_cost_usd` cumulative + list price; 5 process song song không lỗi; `rate_limit_event` có `unifiedWindows` | **Đã xác minh** (Claude 2.1.312, SDK 0.3.314, 2026-10-30) |
| `--bare` không dùng OAuth; Manual mode mặc định ở `-p`; `-p` vẫn nạp settings/`.mcp.json` repo | **Đã xác minh** |
| Codex `codex login status` exit 0 ("Logged in using ChatGPT"); `--full-auto` bị gỡ; `untrusted` retire; bindings app-server 0.164 sinh được | **Đã xác minh** (binary bundled 0.164.0-alpha.11.1) |
| Codex app-server chạy end-to-end trên Windows (approval, thread/resume, sandbox MXC), `Not logged in` exit code, exit code của `exec` | **Giả định** — CLI chưa cài, chưa chạy live |
| Codex giới hạn song song 2–3, mức quota từng model | Giả định (docs + cộng đồng) |
| OpenHands: kiến trúc, API, event schema (đọc code TS) | Đã xác minh bằng đọc code; **chưa chạy server** |
| OpenHands trên Windows native không sandbox chạy ổn với NestJS; ACP trong container | **Giả định** |
| Lệnh cài Claude CLI độc lập; hành vi `.cmd` spawn trên Node 24; Job Object | **Giả định** |
| `APPDATA/LOCALAPPDATA` cần thiết khi `env` whitelist | **Giả định** |
| Claude `interrupt()` ngữ nghĩa chi tiết, `steer` | Đã xác minh `interrupt` có trong `sdk.d.ts`; steer chưa xác minh |

## 22. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| P1 | Một interface 13 op, thay `BrainAdapter`/`AgentAdapter` | FR-081, NFR-101; hai research dùng hai interface khác nhau | Đề xuất |
| P2 | Claude = Agent SDK; raw CLI chỉ fallback | in-process approval/hook, types, version deterministic (D-004) | Đề xuất (theo D-004) |
| P3 | Codex = app-server chính, exec dự phòng | nhiều thread/1 process, approval tương tác, interrupt (D-005) | Đề xuất (theo D-005) |
| P4 | OpenHands = service ngoài, MVP chỉ native + LLM key, ACP tắt | tránh mâu thuẫn D-008, giảm bề mặt | Cần người dùng phê duyệt |
| P5 | Event chuẩn có `seq` + durable/transient | resume/replay rẻ, Office deterministic | Đề xuất |
| P6 | Mock chỉ CI với 6 lớp chặn | bảo vệ nguyên tắc "không dữ liệu giả" | Đề xuất |
| P7 | Không đọc credential, không nhập mật khẩu, login luôn ngoài app (lệnh hiển thị) | D-008, FR-084/085 | Đã chốt theo FR |
| P8 | Không auto-switch provider | tiền + nhất quán là quyết định Owner | Cần người dùng phê duyệt |

## 23. Dependencies

`@anthropic-ai/claude-agent-sdk@^0.3.314` (+ `zod@^4`), `@openai/codex` (global, bản resolve được), `@openai/codex-sdk` (tùy, cho exec fallback), `@openhands/typescript-client@1.59.0` + `uv`/Docker, `@napi-rs/keyring`, RxJS hoặc `AsyncIterable` thuần, `ws`. Phụ thuộc tài liệu: `SECURITY.md` §3–5, `AGENT_SCHEDULER.md`, `RESOURCE_MANAGEMENT.md`, `DATABASE_DESIGN.md` (enum `provider_kind`, `agent_sessions`, `usage_records`), `VIRTUAL_OFFICE.md` (tiêu thụ event), `design/SCREEN_SPECIFICATION.md` §3.11/§3.17.

## 24. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| SDK 0.3.x và app-server `[experimental]` đổi nhanh | pin version, canary đêm, mapper có `default: ignore`, capability thu hẹp theo version |
| Quota subscription dùng chung, ToS nhiều phiên song song/"công ty AI" trên một tài khoản | thu thập số liệu thật, hiển thị gauge, Owner xác nhận điều khoản (DECISION_LOG mâu thuẫn #2) |
| Spawn tax ~48k token/process; đổi `CLAUDE_CONFIG_DIR` riêng có thể làm mất login | thử `settingSources`/`strictMcpConfig` trước; chỉ dùng config dir riêng nếu chứng minh vẫn đăng nhập được **[Giả định]** |
| Tier 1 không phải ranh giới bảo mật (D-007) | policy + hook + diff gate; Tier 2 cho repo không tin cậy |
| Codex chưa test live | checklist §11.4 research, chặn phát hành adapter Codex ở trạng thái "chưa xác minh" trong UI |
| Payload provider chứa secret in ra console | redaction + ephemeral cho task nhạy cảm |
| Hai nguồn sự thật history (CLI vs DB) | DB `agent_events` là chính; transcript CLI chỉ audit/resume |

## 25. Vấn đề chưa giải quyết và mâu thuẫn phát hiện

1. Approval nguồn chính: research Claude chọn `canUseTool`, SECURITY §5.1 chọn `PreToolUse` — hợp nhất thành hai tầng (§17), cần cập nhật SECURITY.
2. Concurrency mặc định: research Claude `MAX_PARALLEL_AGENTS=10`, D-010/RESOURCE = 5, Codex 2–3, DB `provider_configs.max_concurrent` mặc định 2.
3. `provider_kind` enum DB (`claude_code, anthropic_api, openai, gemini, ollama, openrouter, custom`) **thiếu** `codex`, `openhands`, `generic_cli`; cần migration (tôi không sửa file không do mình tạo).
4. RESOURCE §8.1 nói chưa có API đọc quota Claude, trong khi `rate_limit_event.unifiedWindows` đã xác minh; RESOURCE §8.2 coi Codex hoàn toàn giả định trong khi `account/rateLimits/read` có trong bindings.
5. RESOURCE §4.1 cho rằng `WAITING_APPROVAL` không chiếm slot, nhưng approval mức tool giữ process sống (§17).
6. RESOURCE §4.3 đề xuất `CLAUDE_CONFIG_DIR` riêng; chưa chứng minh không mất OAuth login; research Claude chọn `settingSources:['user']`.
7. SECURITY §5.1 dùng tên `exec_approval_request` (v1 legacy), app-server v2 dùng `item/commandExecution/requestApproval`.
8. Env whitelist: SECURITY không có `APPDATA/LOCALAPPDATA`, research Claude yêu cầu giữ.
9. FR-078 "cơ chế chưa rõ" đã lỗi thời; OpenHands ACP vs D-008 (§9.5).
10. Có nên cho phép đăng nhập Codex từ UI (`account/login/start`) trái với SCREEN §3.11?
11. Có chấp nhận API key (FR-098/OI-06) cho Claude/Codex khi quota cạn?
12. Cách dùng `Query.interrupt()` trước `steer` của Claude: chưa xác minh.
13. Claude 2.1.x `PermissionRequest` hook vs `canUseTool`: kiểm tra có trùng lặp quyết định không.

## 26. Acceptance Criteria

- [ ] Một test contract chạy xanh cho Claude, Codex (fixture), OpenHands (fixture), Generic với cùng kịch bản; `seq` liên tục, không event nào thiếu `sessionId/agentId`.
- [ ] 13 operation tồn tại trên mọi adapter; operation không hỗ trợ ném `NOT_SUPPORTED{op}` (test từng ô trống ở §4).
- [ ] `connect → authenticate` phân biệt đúng `NOT_INSTALLED`, `VERSION_UNSUPPORTED`, `NOT_AUTHENTICATED`, `healthy` trên máy dev (Claude) và trên máy chưa cài Codex.
- [ ] Không có code path đọc `.credentials.json`/`auth.json` (grep CI + test); log/WS/DB không chứa secret mẫu (fuzz redaction, kể cả chunk boundary).
- [ ] `cancelTask(soft)` kết thúc turn ≤ 5 s và giữ session resumable; `hard` diệt cả cây process ≤ 15 s (5 session có child giả lập).
- [ ] Vượt `maxUsd` → hủy ≤ 5 s kể từ event usage; cost Claude chỉ phát delta, không cộng đôi sau resume.
- [ ] `approval.requested` xuất hiện ở Approval Center ≤ 1 s; quyết định trả về provider đúng định dạng; đổi lệnh sau duyệt bị chặn bởi `inputHash`.
- [ ] Rate-limit giả lập (mock, CI) đưa pool vào `RATE_LIMITED`, không spawn mới, không auto-switch; trạng thái đổi trên Provider Hub ≤ 2 s.
- [ ] Bundle production không chứa `MockProviderAdapter`; registry từ chối `mock` khi `AIW_TEST_MODE!=='ci'`.
- [ ] Codex adapter đánh dấu "chưa xác minh live" cho tới khi checklist §11.4 research hoàn tất trên máy có CLI cài.
- [ ] Mọi giá trị usage hiển thị có nhãn `list`/`estimated`/`billed`.


> **Phụ lục sửa đổi (D-021, PROPOSED):** nạp skill Markdown động theo từng agent, quét lúc giao task, xem `docs/architecture/DYNAMIC_SKILL_INJECTION.md`. Phụ lục này bổ sung nguồn skill trên đĩa và rào chắn `.ai/**`; phần còn lại của tài liệu này giữ nguyên cho đến khi Owner duyệt.
