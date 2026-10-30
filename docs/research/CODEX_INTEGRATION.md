# Nghiên cứu tích hợp OpenAI Codex CLI vào AI WORKSPACE

> Agent 06 – Codex Integration Researcher. Ngày: 2026-10-30.
> Mục tiêu: dùng Codex CLI làm "bộ não" cho AI employee, điều khiển từ NestJS backend, **tái sử dụng ChatGPT login** (không API key), chạy **nhiều session song song** trên Windows 11.

---

> **[Ghi chú 2026-10-30, OWR-2 (D-005, Q-10)]**: Codex HOÃN khỏi MVP; giữ abstraction `AgentAdapter`; không cài Codex CLI (không chạy `npm i -g @openai/codex`) và không live test ở phạm vi hiện tại. Toàn bộ tài liệu này là nghiên cứu tham khảo, chưa kiểm chứng. Không mặc định subscription ChatGPT cho phép nhiều session song song (OWR-2 mục 5.1).

## 0. Trạng thái máy hiện tại (QUAN TRỌNG)

| Hạng mục | Kết quả kiểm tra |
|---|---|
| `codex` trong PATH | **KHÔNG có** (`Get-Command codex` trả về rỗng) |
| `npm ls -g --depth=0` | chỉ có `pnpm@10.34.6` – **`@openai/codex` chưa cài global** |
| `~/.codex/` | **CÓ** (do VS Code extension `openai.chatgpt` tạo): `auth.json`, `config.toml`, `sessions/`, `session_index.jsonl`, `state_5.sqlite`, `thread_history_1.sqlite`, `logs_2.sqlite`, `.sandbox/`, `skills/`, `plugins/` |
| `codex login status` (chạy bằng binary bundled trong extension) | `Logged in using ChatGPT`, exit code `0` |
| Binary bundled | `C:\Users\TUYEN\.vscode\extensions\openai.chatgpt-26.51021.22027-win32-x64\bin\windows-x86_64\codex.exe` → `codex-cli 0.164.0-alpha.11.1` |
| npm registry | `@openai/codex` latest = **0.166.0**; `@openai/codex-sdk` latest = **0.166.0** (depends on `@openai/codex@0.166.0`) |
| WSL | WSL2 có, default distro = `docker-desktop` (không phải distro dev) |
| `~/.codex/config.toml` hiện tại | `[windows] sandbox = "elevated"` + `[projects.'d:\code\tuyen'] trust_level = "trusted"` |

**Kết luận:** Codex CLI **chưa được cài đặt** như một công cụ dòng lệnh độc lập. Mọi "live test" (`codex exec --json`, `codex app-server`) trong tài liệu này **đang chờ user cài** (`npm i -g @openai/codex`). Tuy nhiên, vì đã có `~/.codex/auth.json` (ChatGPT login từ extension VS Code) và tài liệu chính thức nói **CLI và IDE extension chia sẻ cùng cached login**, nên sau khi cài CLI sẽ **không cần login lại**.

Tôi chỉ dùng binary bundled để đọc `--help`, `--version`, `login status` và `app-server generate-ts/generate-json-schema` (đọc-only, không tạo session). Không đọc nội dung `auth.json`.

---

## 1. Cài đặt trên Windows

### 1.1 Các cách cài

```powershell
# Cách khuyến nghị cho NestJS backend (npm, cài global)
npm install -g @openai/codex
codex --version

# Hoặc cài cục bộ trong project backend (SDK kéo theo CLI)
npm install @openai/codex-sdk     # dependency: @openai/codex@<same version>, có optionalDependencies theo platform
```

- Package `@openai/codex` tải binary native theo platform qua optional dependency (`@openai/codex-win32-x64`, ...). SDK tìm binary tại `node_modules/@openai/codex-win32-x64/vendor/x86_64-pc-windows-msvc/bin/codex.exe` (có file marker `codex-package.json`).
- Trong WSL2: `curl -fsSL https://chatgpt.com/codex/install.sh | sh`.
- Desktop app / VS Code extension cũng bundle `codex.exe` riêng (đã thấy trên máy này) – **không nên** dùng path đó cho backend vì version thay đổi theo extension update.

### 1.2 Native Windows vs WSL

Theo docs chính thức (`learn.chatgpt.com/docs/windows/*`):

- Codex **chạy native trong PowerShell** trên Windows với **Windows sandbox** riêng, **không bắt buộc WSL**.
- WSL2 chỉ được gợi ý khi: cần tooling Linux-native, repo đã sống trong WSL2, hoặc Windows sandbox không đáp ứng. WSL1 **không còn hỗ trợ** (từ 0.115, Linux sandbox chuyển sang `bubblewrap`).
- Repo dưới `/mnt/c` trong WSL chậm; nếu dùng WSL thì giữ repo trong Linux home.
- Windows 11 được khuyến nghị; Windows 10 cần ≥ 1809 (ConPTY), best-effort.

**Khuyến nghị cho AI WORKSPACE:** chạy **native Windows** (NestJS spawn `codex.exe` trực tiếp) để dùng chung `~/.codex/auth.json` với VS Code extension và tránh ranh giới WSL/Windows path.

---

## 2. Authentication

### 2.1 Lệnh

```text
codex login                     # ChatGPT OAuth qua browser, callback http://localhost:1455
codex login --device-auth       # Device code flow (beta) – cho headless/remote; phải bật trong ChatGPT security settings
printenv OPENAI_API_KEY | codex login --with-api-key        # API key qua stdin
printenv CODEX_ACCESS_TOKEN | codex login --with-access-token  # Enterprise access token
codex login status              # In "Logged in using ChatGPT" | "Logged in using API key" | "Not logged in"
codex logout
```

- `codex login status` trả về **exit 0** khi đã login (đã xác nhận trên máy này: `Logged in using ChatGPT`). Khi chưa login, in `Not logged in` và exit ≠ 0 (cần xác nhận lại sau khi cài, chưa test được).
- Token ChatGPT được **tự động refresh** trước khi hết hạn.

### 2.2 Nơi lưu

- `$CODEX_HOME/auth.json` (mặc định `~/.codex/auth.json`), **plaintext** – docs nói "treat like a password".
- `cli_auth_credentials_store = "file" | "keyring" | "auto" | "ephemeral"` trong `config.toml`.
- **CLI + IDE extension + Desktop app dùng chung** cache login. Logout ở một nơi thì nơi khác phải login lại.

### 2.3 Dùng API key thay thế

- `CODEX_API_KEY=<key> codex exec ...` cho một lần chạy (không ghi vào auth.json). SDK TypeScript inject `CODEX_API_KEY` khi truyền `apiKey`.
- Billing theo giá API chuẩn, không trừ quota ChatGPT plan.
- Admin có thể ép: `forced_login_method = "chatgpt" | "api"`, `forced_chatgpt_workspace_id`.

### 2.4 Multi-account / nhiều `CODEX_HOME`

Mỗi process đọc `CODEX_HOME`. Có thể chạy từng agent với `CODEX_HOME` riêng (mỗi thư mục có `auth.json` riêng) nếu muốn tách tài khoản/plan – nhưng mặc định AI WORKSPACE dùng chung một `~/.codex`.

### 2.5 App Server auth API

```json
{ "method": "account/read", "id": 1, "params": { "refreshToken": false } }
{ "id": 1, "result": { "account": { "type": "chatgpt", "email": "...", "planType": "plus" }, "requiresOpenaiAuth": true } }

{ "method": "account/login/start", "id": 3, "params": { "type": "chatgpt" } }
{ "id": 3, "result": { "type": "chatgpt", "loginId": "<uuid>", "authUrl": "https://..." } }
{ "method": "account/login/completed", "params": { "loginId": "<uuid>", "success": true, "error": null } }
{ "method": "account/updated", "params": { "authMode": "chatgpt", "planType": "plus" } }

{ "method": "account/rateLimits/read", "id": 6 }
```

`AuthMode = "apikey" | "chatgpt" | "chatgptAuthTokens" | "headers" | "agentIdentity" | "personalAccessToken" | "bedrockApiKey" | "bedrockAccessKeys"`.

→ Backend có thể hiển thị trạng thái login và trigger login flow (mở `authUrl` cho user) ngay từ UI, không cần terminal.

---

## 3. Non-interactive: `codex exec`

### 3.1 Cú pháp và flags (từ `codex exec --help`, v0.164)

```text
codex exec [OPTIONS] [PROMPT]
codex exec resume [OPTIONS] [SESSION_ID] [PROMPT]     # hoặc --last
codex exec fork <SESSION_ID> <PROMPT>
codex exec review

  -C, --cd <DIR>                       working root
  -s, --sandbox <read-only|workspace-write|danger-full-access>
      --approve-for-me                 route approvals qua auto-review + workspace-write
      --dangerously-bypass-approvals-and-sandbox   (alias --yolo) – KHÔNG khuyến nghị
  -m, --model <MODEL>
  -c, --config key=value               override config.toml (value parse TOML), lặp được
  -p, --profile <NAME>                 layer $CODEX_HOME/<name>.config.toml
      --add-dir <DIR>                  thêm thư mục writable
      --skip-git-repo-check            cho phép chạy ngoài git repo
      --ephemeral                      không ghi rollout ra đĩa
      --ignore-user-config             bỏ qua $CODEX_HOME/config.toml (auth vẫn dùng CODEX_HOME)
      --output-schema <FILE>           JSON Schema cho final response
      --json                           JSONL events ra stdout
  -o, --output-last-message <FILE>     ghi message cuối ra file
  -i, --image <FILE>...
      --color always|never|auto
      --worktree                       chạy trong git worktree mới (managed)
      --thread-source <SOURCE>
      --enable/--disable <FEATURE>
```

**Khác biệt quan trọng so với tài liệu cũ (đã kiểm chứng trên binary 0.164):**

- `--full-auto` → **error: unexpected argument** (docs: "No longer supported. Use `--sandbox workspace-write`").
- `-a/--ask-for-approval` **không còn** trên `codex exec` (chỉ còn trên TUI, với 2 giá trị `on-request | never`). Với `exec` phải set qua `-c approval_policy="never"`.
- `approval_policy = "untrusted"` **bị retire** (có thể làm client không khởi động); `on-failure` **deprecated**. Giá trị hợp lệ: `on-request`, `never`, hoặc bảng `granular`.
- Prompt: argument, hoặc `-` để đọc stdin; nếu vừa có arg vừa pipe stdin thì stdin được append thành block `<stdin>`.
- Mặc định `exec` yêu cầu cwd là git repo → với workspace của AI employee cần `--skip-git-repo-check` hoặc `git init`.

### 3.2 Lệnh mẫu cho backend

```powershell
codex exec --json `
  -C "D:\workspaces\emp-01\repo" `
  --sandbox workspace-write `
  -c approval_policy="never" `
  -c model_reasoning_effort="medium" `
  --skip-git-repo-check `
  -o "D:\workspaces\emp-01\last.md" `
  "Implement feature X, run tests, summarize."

# tiếp tục thread
codex exec --json resume 01a15f57-8363-78c2-b75c-106f978c7a1f "Now add unit tests"
codex exec --json resume --last "continue"
```

### 3.3 Định dạng JSONL (`--json`)

Mỗi dòng stdout là một JSON object. Event types (docs + SDK `events.ts`):

```jsonl
{"type":"thread.started","thread_id":"01a1..."}
{"type":"turn.started"}
{"type":"item.started","item":{"id":"item_1","type":"command_execution","command":"bash -lc ls","status":"in_progress"}}
{"type":"item.updated","item":{...}}
{"type":"item.completed","item":{"id":"item_1","type":"command_execution","command":"...","aggregated_output":"...","exit_code":0,"status":"completed"}}
{"type":"item.completed","item":{"id":"item_2","type":"agent_message","text":"Done."}}
{"type":"turn.completed","usage":{"input_tokens":24763,"cached_input_tokens":24448,"cache_write_input_tokens":0,"output_tokens":122,"reasoning_output_tokens":0}}
{"type":"turn.failed","error":{"message":"..."}}
{"type":"error","message":"..."}
```

Item types (`ThreadItem` trong SDK `items.ts`):

| `item.type` | Fields chính |
|---|---|
| `agent_message` | `text` (JSON string nếu dùng `--output-schema`) |
| `reasoning` | `text` (summary) |
| `command_execution` | `command`, `aggregated_output`, `exit_code?`, `status: in_progress\|completed\|failed` |
| `file_change` | `changes: [{path, kind: add\|delete\|update}]`, `status: completed\|failed` |
| `mcp_tool_call` | `server`, `tool`, `arguments`, `result?: {content, structured_content, _meta}`, `error?: {message}`, `status` |
| `web_search` | `query` |
| `todo_list` | `items: [{text, completed}]` |
| `error` | `message` (non-fatal) |

Lưu ý: stderr chứa progress log; stdout chỉ có JSONL (khi `--json`) hoặc final message (khi không). Exit code ≠ 0 khi lỗi (docs không liệt kê bảng exit code cụ thể; cần đo sau khi cài).

### 3.4 `--output-schema`

```json
{ "type": "object", "properties": { "project_name": {"type":"string"}, "programming_languages": {"type":"array","items":{"type":"string"}} }, "required": ["project_name","programming_languages"], "additionalProperties": false }
```

Final `agent_message.text` sẽ là JSON hợp lệ theo schema; kết hợp `-o file.json` để đọc kết quả có cấu trúc (dùng cho task "trả về plan/estimate dạng JSON").

---

## 4. Codex App Server (`codex app-server`) – ĐƯỜNG TÍCH HỢP CHÍNH

### 4.1 Transport & framing

```text
codex app-server                      # stdio (mặc định, --listen stdio://), JSONL: 1 message/dòng
codex app-server --listen ws://127.0.0.1:4500   # WebSocket (experimental), 1 JSON-RPC message/text frame
   --ws-auth capability-token --ws-token-file <abs path> | --ws-token-sha256 <hex>
   --ws-auth signed-bearer-token --ws-shared-secret-file <abs> [--ws-issuer --ws-audience --ws-max-clock-skew-seconds]
codex app-server --listen unix://PATH
codex app-server generate-ts --out ./schemas [--experimental]
codex app-server generate-json-schema --out ./schemas [--experimental]
codex app-server --print-startup-config
codex app-server daemon ...           # shared local daemon; TUI có --no-daemon
```

- JSON-RPC 2.0 nhưng **bỏ field `"jsonrpc":"2.0"`** trên wire.
- Request: `{method, id, params}`; Response: `{id, result}` hoặc `{id, error:{code,message}}`; Notification: không có `id`.
- Server → client **requests** (có `id`, client phải trả lời): approvals, user input, elicitation.
- WS listener có `GET /readyz`, `GET /healthz`; request có header `Origin` → 403.
- Error `-32001` = server overloaded → retry backoff + jitter. `-32602` invalid params.
- Subcommand vẫn gắn nhãn `[experimental]` trong help (0.164) nhưng là thứ VS Code extension/Desktop app dùng hằng ngày.

Tôi đã generate bindings từ binary 0.164 ra: `C:\Users\TUYEN\AppData\Local\Temp\claude\d--Code-Tuyen-Agent-management\9dde48b7-c365-4f17-9f11-5331b00699a6\scratchpad\schema\{ts,json}` (1362 file). Sau khi cài CLI, nên chạy lại `generate-ts` và commit vào `packages/codex-protocol/` để type-safe và khớp đúng version.

### 4.2 Handshake

```jsonl
{"method":"initialize","id":0,"params":{"clientInfo":{"name":"ai_workspace","title":"AI Workspace","version":"0.1.0"},"capabilities":{"experimentalApi":true,"optOutNotificationMethods":[]}}}
{"id":0,"result":{"userAgent":"codex_cli_rs/0.164.0 ...","codexHome":"C:\\Users\\TUYEN\\.codex","platformFamily":"windows","platformOs":"windows"}}
{"method":"initialized","params":{}}
```

- Gọi `initialize` đúng 1 lần/connection; trước đó mọi request khác → `Not initialized`.
- `experimentalApi: true` cần cho các method/field experimental (vd `experimentalRawEvents`).

### 4.3 Thread lifecycle

```jsonl
{"method":"thread/start","id":10,"params":{"cwd":"D:\\workspaces\\emp-01\\repo","model":"gpt-6.1-sol","approvalPolicy":"on-request","sandbox":"workspace-write","serviceName":"ai_workspace","developerInstructions":"You are Backend Engineer #1 ...","ephemeral":false}}
{"id":10,"result":{"thread":{"id":"01a1...","sessionId":"...","cwd":"...","status":{...},"createdAt":...},"model":"gpt-6.1-sol","modelProvider":"openai","approvalPolicy":"on-request","sandbox":{"type":"workspaceWrite","writableRoots":["D:\\workspaces\\emp-01\\repo"],"networkAccess":false,"excludeTmpdirEnvVar":false,"excludeSlashTmp":false},"reasoningEffort":"medium",...}}
{"method":"thread/started","params":{"thread":{...}}}

{"method":"thread/resume","id":11,"params":{"threadId":"01a1...","excludeTurns":true}}
{"method":"thread/list","id":12,"params":{"cwd":"D:\\workspaces\\emp-01\\repo","limit":20,"sortKey":"updatedAt"}}
{"method":"thread/fork","id":13,"params":{"threadId":"01a1...","ephemeral":true}}
{"method":"thread/archive","id":14,"params":{"threadId":"01a1..."}}
{"method":"thread/read","id":15,"params":{"threadId":"01a1..."}}
```

`ThreadStartParams` (generated, rút gọn): `model?, modelProvider?, cwd?, runtimeWorkspaceRoots?, approvalPolicy?: AskForApproval, approvalsReviewer?, sandbox?: SandboxMode, permissions?: string (profile name), config?: {[key]: JsonValue}, serviceName?, baseInstructions?, developerInstructions?, personality?, multiAgentMode?, ephemeral?, historyMode?, threadSource?, projectId?, environments?, dynamicTools?, experimentalRawEvents?`.

→ `developerInstructions` + `config` per-thread cho phép **mỗi AI employee có persona/config riêng trên cùng một app-server process**.

### 4.4 Turn lifecycle

```jsonl
{"method":"turn/start","id":30,"params":{"threadId":"01a1...","input":[{"type":"text","text":"Run tests and fix failures"},{"type":"localImage","path":"D:\\tmp\\shot.png"}],"cwd":"D:\\workspaces\\emp-01\\repo","approvalPolicy":"on-request","sandboxPolicy":{"type":"workspaceWrite","writableRoots":["D:\\workspaces\\emp-01\\repo"],"networkAccess":true,"excludeTmpdirEnvVar":false,"excludeSlashTmp":false},"model":"gpt-6.1-sol","effort":"medium","summary":"concise","outputSchema":{"type":"object"}}}
{"id":30,"result":{"turn":{"id":"turn_...","status":"inProgress","items":[],"error":null}}}
{"method":"turn/started","params":{"threadId":"...","turn":{...}}}
... notifications ...
{"method":"turn/completed","params":{"threadId":"...","turn":{"id":"...","status":"completed|interrupted|failed","items":[...],"error":null}}}

{"method":"turn/interrupt","id":31,"params":{"threadId":"...","turnId":"..."}}      -> {}
{"method":"turn/steer","id":32,"params":{"threadId":"...","input":[...],"expectedTurnId":"..."}}  -> {"turnId":"..."}
```

Input types: `text`, `image` (url), `localImage` (path), `skill`.
`TurnStatus = "completed" | "interrupted" | "failed" | "inProgress"`; `TurnError = {message, codexErrorInfo, additionalDetails, misalignment}`.

### 4.5 Notifications cần subscribe (từ `ServerNotification.ts`)

```text
thread/started, thread/status/changed, thread/closed, thread/name/updated, thread/tokenUsage/updated, thread/compacted
turn/started, turn/completed, turn/diff/updated, turn/plan/updated
item/started, item/completed
item/agentMessage/delta            { threadId, turnId, itemId, delta }
item/plan/delta, item/reasoning/summaryTextDelta, item/reasoning/textDelta
item/commandExecution/outputDelta, item/fileChange/outputDelta, item/fileChange/patchUpdated
item/mcpToolCall/progress
item/autoApprovalReview/started|completed
serverRequest/resolved             { threadId, requestId }
error                              { error: TurnError, willRetry, threadId, turnId }
warning, configWarning, deprecationNotice, model/rerouted
account/updated, account/rateLimits/updated, account/login/completed
windowsSandbox/setupCompleted, windows/worldWritableWarning
mcpServer/startupStatus/updated, fs/changed
```

Payloads:

```ts
ItemStartedNotification   = { item: ThreadItem, threadId, turnId, startedAtMs }
ItemCompletedNotification = { item: ThreadItem, threadId, turnId, completedAtMs }   // authoritative state
ThreadTokenUsageUpdatedNotification = { threadId, turnId, tokenUsage: { total: TokenUsageBreakdown, last: TokenUsageBreakdown, modelContextWindow } }
```

`ThreadItem` (v2, camelCase – KHÁC snake_case của exec --json):
`userMessage | agentMessage{text, phase: commentary|partial_answer|final_answer} | plan{text} | reasoning{summary[], content[]} | commandExecution{command, cwd, status, commandActions[], aggregatedOutput, exitCode, durationMs, processId} | fileChange{changes[{path,kind,diff}], status} | mcpToolCall{server, tool, status, arguments, result, error, durationMs} | dynamicToolCall | collabAgentToolCall | subAgentActivity | webSearch | imageView | imageGeneration | enteredReviewMode | exitedReviewMode | contextCompaction | functionCallOutput | hookPrompt | sleep`

### 4.6 Approval requests (server → client, PHẢI trả lời)

```ts
// ServerRequest union (generated)
"item/commandExecution/requestApproval"  params: CommandExecutionRequestApprovalParams
"item/fileChange/requestApproval"        params: FileChangeRequestApprovalParams
"item/permissions/requestApproval"       params: PermissionsRequestApprovalParams
"item/tool/requestUserInput"             params: ToolRequestUserInputParams
"mcpServer/elicitation/request"
"item/tool/call"                         (dynamic tools do client cung cấp)
"account/chatgptAuthTokens/refresh", "attestation/generate", "currentTime/read"
"execCommandApproval", "applyPatchApproval"   (legacy v1)
```

```jsonl
← {"method":"item/commandExecution/requestApproval","id":77,"params":{"kind":"...","threadId":"...","turnId":"...","itemId":"item_9","startedAtMs":1730,"reason":"needs network","command":"npm install","cwd":"D:\\...","commandActions":[],"networkApprovalContext":{"host":"registry.npmjs.org","protocol":"https"},"availableDecisions":["accept","acceptForSession","decline","cancel"],"proposedExecpolicyAmendment":null}}
→ {"id":77,"result":{"decision":"acceptForSession"}}

← {"method":"item/fileChange/requestApproval","id":78,"params":{"threadId":"...","turnId":"...","itemId":"item_10","startedAtMs":1730,"reason":"write outside workspace","grantRoot":"D:\\other"}}
→ {"id":78,"result":{"decision":"accept"}}

← {"method":"item/permissions/requestApproval","id":79,"params":{"threadId":"...","turnId":"...","itemId":"...","environmentId":null,"startedAtMs":1730,"cwd":"...","reason":"...","permissions":{...RequestPermissionProfile}}}
→ {"id":79,"result":{"permissions":{...GrantedPermissionProfile},"scope":"session"}}   // scope: "turn" | "session"

← {"method":"serverRequest/resolved","params":{"threadId":"...","requestId":77}}
```

Decision enums:

```ts
CommandExecutionApprovalDecision = "accept" | "acceptForSession" | "decline" | "cancel"
  | { acceptWithExecpolicyAmendment: { execpolicy_amendment } }
  | { applyNetworkPolicyAmendment: { network_policy_amendment } }
FileChangeApprovalDecision = "accept" | "acceptForSession" | "decline" | "cancel"
AskForApproval = "untrusted" | "on-request" | "never" | { granular: { sandbox_approval, rules, skill_approval, request_permissions, mcp_elicitations } }
SandboxPolicy = {type:"dangerFullAccess"} | {type:"readOnly", networkAccess} | {type:"workspaceWrite", writableRoots[], networkAccess, excludeTmpdirEnvVar, excludeSlashTmp} | {type:"externalSandbox", networkAccess}
```

Nếu turn start/complete/interrupt xảy ra trước khi client trả lời, server tự clear request và gửi `serverRequest/resolved`.

→ Đây chính là **human-in-the-loop** của AI WORKSPACE: backend nhận request → đẩy lên UI (WebSocket tới frontend) → manager người thật bấm Approve/Decline → backend trả `result` về app-server. Có thể cài **policy engine** tự động accept cho lệnh whitelist, hoặc dùng `approvalsReviewer: "auto_review"` để một reviewer agent của Codex tự duyệt.

### 4.7 Lợi thế App Server so với exec

| | `app-server` | `exec --json` |
|---|---|---|
| Nhiều thread trên 1 process | **Có** (threadId trong mọi message) | 1 process = 1 turn |
| Approval tương tác | **Có** (request/response) | Không (chỉ `never`/auto-review) |
| Streaming delta token | **Có** (`item/agentMessage/delta`) | Chỉ item-level |
| Interrupt / steer giữa turn | **Có** | Kill process |
| Đọc account/rate limit/model list | **Có** (`account/*`, `model/list`) | Không |
| Login flow từ UI | **Có** (`account/login/start`) | Không |
| Ổn định API | `[experimental]`, bindings generate theo version | Ổn định hơn, snake_case |

---

## 5. `@openai/codex-sdk` (TypeScript)

```ts
import { Codex } from "@openai/codex-sdk";

const codex = new Codex({
  codexPathOverride?: string,   // path tới codex.exe; mặc định resolve từ @openai/codex-win32-x64 bundled
  apiKey?: string,              // => env CODEX_API_KEY cho child (KHÔNG cần nếu dùng ChatGPT login)
  baseUrl?: string,             // => --config openai_base_url=...
  config?: object,              // flatten thành --config a.b=c (TOML literal)
  configOverrides?: string[],   // raw "key=value" TOML, apply sau config
  env?: Record<string,string>,  // nếu truyền thì KHÔNG inherit process.env
});

const thread = codex.startThread({
  model?, sandboxMode?: "read-only"|"workspace-write"|"danger-full-access",
  workingDirectory?, skipGitRepoCheck?: boolean,
  modelReasoningEffort?: "minimal"|"low"|"medium"|"high"|"xhigh"|"max"|"ultra"|"persistent",
  networkAccessEnabled?: boolean, webSearchMode?: "disabled"|"cached"|"live", webSearchEnabled?: boolean,
  approvalPolicy?: "never"|"on-request"|"on-failure"|"untrusted",   // SDK type còn giữ 2 giá trị cũ; CLI mới reject "untrusted"
  additionalDirectories?: string[], threadSource?: string,
});
const r = await thread.run("prompt" | [{type:"text",text}, {type:"local_image",path}], { outputSchema? });
r.finalResponse; r.items; thread.id;             // thread.id có sau thread.started

const { events } = await thread.runStreamed("prompt");
for await (const ev of events) { /* ThreadEvent: thread.started | turn.started | turn.completed{usage} | turn.failed{error} | item.started | item.updated | item.completed | error */ }

const t2 = codex.resumeThread("<thread-id>");   // đọc từ ~/.codex/sessions
```

Cách SDK spawn (đọc từ `sdk/typescript/src/exec.ts`):

```text
codex exec --experimental-json [--config k=v]* [--model m] --sandbox <mode> --cd <dir> [--add-dir d]* [--skip-git-repo-check] [--output-schema f] --config model_reasoning_effort="..." [--config sandbox_workspace_write.network_access=bool] --config web_search="..." --config approval_policy="..." [resume <threadId>] [--image p]*
prompt → stdin (đóng sau khi ghi); stdout đọc theo dòng; stderr buffer, chỉ trả khi exit ≠ 0 ("Codex Exec exited with code N")
env: CODEX_INTERNAL_ORIGINATOR_OVERRIDE=codex_sdk_ts, CODEX_API_KEY nếu có apiKey
```

**Có tái sử dụng ChatGPT login không?** **Có.** SDK chỉ là wrapper spawn `codex exec`; nếu không truyền `apiKey`, child process đọc `$CODEX_HOME/auth.json` như CLI bình thường. Docs `non-interactive-mode` xác nhận: "By default, `codex exec` reuses saved CLI credentials".

**Hạn chế SDK:** không có approval callback (chỉ phù hợp `approval_policy=never`), không interrupt giữa chừng ngoài AbortSignal (kill process), không streaming delta, mỗi `run()` = 1 process mới (resume tốn thời gian load history). → SDK chỉ nên là **fallback / batch mode**.

---

## 6. MCP

### 6.1 Codex là MCP client (`config.toml`)

```toml
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]
env = { API_KEY = "..." }
startup_timeout_sec = 10
tool_timeout_sec = 60
enabled = true
required = false
enabled_tools = ["resolve-library-id"]
default_tools_approval_mode = "auto"   # auto | prompt | writes | approve

[mcp_servers.ai_workspace]            # streamable HTTP – backend NestJS expose MCP
url = "http://127.0.0.1:3000/mcp"
bearer_token_env_var = "AIW_TOKEN"
http_headers = { "X-Employee-Id" = "emp-01" }
```

CLI: `codex mcp add <name> [--env K=V] -- <cmd> [args]`, `codex mcp add <name> --url <url> --oauth-client-id ...`, `codex mcp list|get|remove|login|logout`. Project-local `.codex/config.toml` chỉ được đọc khi project `trusted`.

→ AI WORKSPACE nên expose một **MCP server HTTP** từ NestJS (task board, ticket, memory, message bus) để mọi AI employee (Codex lẫn Claude Code) gọi cùng một tool set.

### 6.2 Codex là MCP server

`codex mcp-server` vẫn tồn tại (hidden subcommand, `codex mcp-server -V` trả version) nhưng **không còn trong `--help` và không còn trên docs** (docs MCP hiện chỉ nói về Codex làm client). Tool cũ `codex` / `codex-reply`. **Không nên** dựa vào đường này; App Server là thay thế chính thức.

---

## 7. `config.toml`, profiles, model, sandbox Windows

### 7.1 Keys chính

```toml
model = "gpt-6.1-sol"                  # hoặc gpt-6.1-terra / gpt-6-luna (rẻ, limit cao) – xem /model trong TUI hoặc model/list
model_provider = "openai"
model_reasoning_effort = "medium"      # minimal|low|medium|high|xhigh|...
approval_policy = "on-request"         # on-request | never | { granular = {...} }   (untrusted: retired; on-failure: deprecated)
approvals_reviewer = "user"            # "auto_review" để Codex tự duyệt
sandbox_mode = "workspace-write"       # read-only | workspace-write | danger-full-access
[sandbox_workspace_write]
writable_roots = ["D:\\workspaces\\shared"]
network_access = false
web_search = "cached"                  # disabled | cached | indexed | live
history.persistence = "save-all"       # none để tắt rollout
[shell_environment_policy]
inherit = "core"                       # all | core | none
[projects.'d:\workspaces\emp-01\repo']
trust_level = "trusted"
[features]
prefer_mxc = true                      # Windows: ưu tiên MXC sandbox
[windows]
sandbox = "elevated"                   # mxc | elevated | unelevated
notify = ["powershell", "-File", "notify.ps1"]   # hook ngoài khi turn xong
cli_auth_credentials_store = "file"
[agents]
max_concurrent_threads_per_session = 2   # legacy alias agents.max_threads – sub-agent trong 1 session
```

Permissions mới (thay cho `sandbox_mode`, **không kết hợp** cả hai): `default_permissions = ":workspace"` hoặc `[permissions.<name>]` với `extends = ":workspace"`, `[permissions.<name>.filesystem]`, `[permissions.<name>.network] enabled = true`. App Server/exec có tham số `permissions: "<profile>"`.

### 7.2 Profiles

Codex 0.16x: `-p/--profile <name>` **layer file `$CODEX_HOME/<name>.config.toml`** lên config gốc (không còn bảng `[profiles.x]` theo help). → Tạo `~/.codex/aiw-backend.config.toml`, `~/.codex/aiw-reviewer.config.toml` cho từng vai trò, hoặc đơn giản hơn: truyền `config` per-thread qua App Server.

### 7.3 Sandbox trên Windows (không có seatbelt/landlock)

| Impl | Yêu cầu | Cơ chế |
|---|---|---|
| `mxc` | Windows 11 24H2 build ≥ 26100.9278 hoặc 25H2 ≥ 26200.9278; **không cần admin** | Native MXC capability; child process bị kill khi foreground command thoát (dev server detach không chạy được) |
| `elevated` | Setup 1 lần cần **UAC/admin** (`codex sandbox setup --elevated ...`) | Tạo sandbox user ít quyền, ACL filesystem, firewall rule, private desktop |
| `unelevated` | Không cần admin | Restricted token từ user hiện tại, ACL; **network isolation yếu**, không hỗ trợ deny-read |

- Máy này đã có `[windows] sandbox = "elevated"` và `~/.codex/.sandbox/` (setup đã chạy qua VS Code extension). OS build 26300 > 26100.9278 → **đủ điều kiện MXC**; có thể thử `codex -c windows.sandbox=mxc sandbox -- cmd /c echo hi` sau khi cài.
- `features.prefer_mxc` mặc định **tắt** trong standalone CLI → phải bật tay nếu muốn.
- Sandbox log: `CODEX_HOME/.sandbox/sandbox.log`. Không gửi `.sandbox-secrets/`.
- Trong `workspace-write`, `.git`, `.codex`, `.agents`, `.aws` luôn read-only; network off trừ khi bật.
- `danger-full-access` = không sandbox, chỉ dùng khi workspace đã được cô lập ngoài (VM/container).

---

## 8. Session rollouts & history (`~/.codex/sessions`)

Cấu trúc thực tế trên máy này (5 session từ VS Code extension):

```text
~/.codex/sessions/YYYY/MM/DD/rollout-<ISO time>-<thread uuid>.jsonl
~/.codex/session_index.jsonl         {"id","thread_name","updated_at"} mỗi dòng
~/.codex/state_5.sqlite, thread_history_1.sqlite, logs_2.sqlite   (index/paginated history – "migrate-rollouts")
```

Mỗi dòng rollout: `{"timestamp","ordinal","type","payload"}` với `type ∈ session_meta | turn_context | response_item | event_msg`:

```jsonl
{"type":"session_meta","payload":{"id":"<thread>","cwd":"d:\\Code\\Tuyen","originator":"codex_vscode","cli_version":"0.162.0-alpha.17.2","source":"vscode","model_provider":"openai","base_instructions":{...}}}
{"type":"turn_context","payload":{"turn_id":"...","cwd":"...","approval_policy":"never","sandbox_policy":{"type":"danger-full-access"},"model":"gpt-6-luna","collaboration_mode":{...}}}
{"type":"event_msg","payload":{"type":"task_started","turn_id":"...","model_context_window":258400}}
{"type":"event_msg","payload":{"type":"item_completed","thread_id":"...","turn_id":"...","item":{"type":"AgentMessage","id":"item-2","content":[{"type":"Text","text":"..."}],"phase":"commentary"}}}
{"type":"response_item","payload":{"type":"custom_tool_call","name":"exec","call_id":"...","input":"..."}}
{"type":"response_item","payload":{"type":"custom_tool_call_output","call_id":"...","output":"..."}}
{"type":"event_msg","payload":{"type":"token_count","info":{"total_token_usage":{...},"last_token_usage":{...},"model_context_window":258400},"rate_limits":{"limit_id":"codex","primary":{"used_percent":14.0,"window_minutes":43200,"resets_at":1795148436},"secondary":null,"credits":{...}}}}
{"type":"event_msg","payload":{"type":"task_complete","turn_id":"...","last_agent_message":"..."}}
```

Ghi chú:
- `event_msg.item_completed.item.type` ở rollout dùng **PascalCase** (`AgentMessage`, `UserMessage`) – format thứ 3 bên cạnh snake_case (exec) và camelCase (app-server). Không nên parse rollout làm nguồn chính; chỉ dùng cho **history/replay/audit**.
- `token_count.rate_limits` trong rollout là nguồn hữu ích để đọc **% quota** đã dùng (window 43200 phút = 30 ngày/… tuỳ plan).
- Pixel Agents (`D:\Code\Tuyen\Agent management\pixel-agents`) hiện **chưa có Codex provider** – `core/src/provider.ts` ghi rõ "Only HookProvider ships today (Claude Code). Transcript-polling ... will be added when a real second provider (Codex ...) lands". `server/src/fileWatcher.ts` có comment "File-based provider (Claude, Codex): adopt with JSONL file watching" → ý tưởng là watch `~/.codex/sessions/**/*.jsonl` bằng chokidar. Có thể dùng cách này cho **observability của session do người dùng chạy ngoài platform** (VS Code), còn session do platform tạo thì đã có stream từ app-server.
- `--ephemeral` (exec) / `ephemeral: true` (thread/start) để **không ghi rollout** cho các turn tạm.

---

## 9. Concurrency & rate limits (ChatGPT plan)

- **Kỹ thuật:** một `codex app-server` process phục vụ **nhiều thread đồng thời**; mỗi message mang `threadId`/`turnId`. Có thể chạy thêm nhiều process `exec` song song – không có khoá file toàn cục ngoài `thread-writer-locks/` per-thread. Không thấy giới hạn số thread trong docs. `agents.max_concurrent_threads_per_session` chỉ giới hạn **sub-agent** nội bộ một session.
- **Quota:** tất cả session (CLI, IDE, cloud, Desktop) **chia sẻ chung** limit của tài khoản: cửa sổ **5 giờ** + **tuần** (Plus/Business); **Pro hiện không có limit 5 giờ**. Ước lượng chính thức (Plus/Business, local messages/5h): GPT-6 Astra 5–45, GPT-6.1 Sol 15–160, GPT-6.1 Terra 35–285, GPT-6 Luna **350–3,000**. Fast mode ×2.5, Ultrafast ×8.
- Hết quota giữa turn: Codex có thể hoàn thành turn đang chạy; sau đó request bị từ chối → `error` notification / `turn.failed` với `codexErrorInfo`. Khi đạt limit là **hard ceiling** (retry vô ích đến khi reset), khác API key.
- Nguồn cộng đồng: với Plus, thực tế ~2 agent song song là mức hợp lý; 4–5 stream đồng thời sẽ đốt 5h window rất nhanh ([danielvaughan](https://codex.danielvaughan.com/2026/04/12/codex-cli-rate-limiting-backoff-retry-quota/), [inventivehq](https://inventivehq.com/blog/codex-cli-usage-rate-limits)).
- **Chiến lược cho platform:**
  1. Đọc `account/rateLimits/read` (hoặc notification `account/rateLimits/updated`) → hiển thị gauge, **throttle scheduler** khi `usedPercent > 80`.
  2. Gán model rẻ (`gpt-6-luna`) cho employee "junior"/task lặp, model mạnh cho architect/reviewer.
  3. Queue task với concurrency cap cấu hình (mặc định 2–3 turn song song/account trong nghiên cứu; giá trị áp dụng: cap provider Codex 3, global 5, theo D-010 APPROVED), backoff + jitter khi `-32001` hoặc rate-limit error.
  4. Cho phép employee cụ thể chạy bằng `CODEX_API_KEY` (billing API) khi quota plan cạn.
  5. ToS: ChatGPT plan là cho cá nhân; chạy "công ty AI" nhiều agent trên 1 tài khoản Plus có rủi ro fair-use – nên cân nhắc Business/Enterprise hoặc API key cho production.

---

## 10. Security notes

1. `auth.json` là plaintext token ChatGPT → backend **không đọc/copy** file này; chỉ spawn `codex` với `CODEX_HOME` đúng. Không bao giờ log env của child. Hạn chế ACL thư mục `~/.codex`.
2. Không để backend chạy mặc định `danger-full-access`/`--yolo`. Dùng `workspace-write` + `approval_policy="on-request"` qua App Server để **human/policy approval**. `never` chỉ cho task read-only hoặc workspace đã cô lập (worktree/container).
3. Windows `unelevated` sandbox cách ly network yếu → nếu không thể MXC/elevated, hãy coi như không có sandbox và bù bằng: workspace riêng per employee (`--worktree` hoặc thư mục riêng), account Windows ít quyền chạy NestJS, firewall.
4. Prompt injection: tắt `web_search` live (giữ `cached`/`disabled`), MCP server chỉ allowlist tool, `default_tools_approval_mode = "writes"`.
5. `.git`, `.codex`, `.agents` read-only trong workspace-write – đừng cho agent tự sửa `config.toml`/hooks. Hooks cần `features.hooks` + trust; `--dangerously-bypass-hook-trust` không dùng.
6. WebSocket app-server: luôn bật `--ws-auth`, bind `127.0.0.1`; stdio là lựa chọn an toàn hơn cho local.
7. Rollout chứa toàn bộ output lệnh (có thể chứa secret in ra console) → cân nhắc `history.persistence = "none"` hoặc `ephemeral` cho task nhạy cảm; hoặc redaction khi index vào DB platform.
8. Mỗi message từ Codex (agentMessage, command output) là **untrusted data** với backend: không eval, escape khi render.

---

## 11. Đề xuất thiết kế `CodexAdapter`

### 11.1 Nguyên tắc

- **Primary: App Server (stdio)** – 1 process `codex app-server` dài hạn per backend node (hoặc per account/`CODEX_HOME`), nhiều thread = nhiều AI employee. Supervisor restart khi process chết; threads resume bằng `thread/resume`.
- **Fallback: `codex exec --json`** – khi app-server lỗi/không tương thích version, hoặc cho job batch một-lần (`approval_policy=never`, `--output-schema`). Có thể dùng `@openai/codex-sdk` làm implementation của fallback để tận dụng parse sẵn.
- **Auxiliary: rollout watcher** – chokidar trên `~/.codex/sessions/**/*.jsonl` để adopt session ngoài platform (giống Pixel Agents), chỉ cho observability.
- Chuẩn hoá tất cả về **AgentEvent** chung với ClaudeAdapter (Agent 05) để UI/scheduler không biết CLI nào.

### 11.2 Interface sketch (TypeScript / NestJS)

> Ghi chú: AgentBrain là tên nghiên cứu; tên chính thức là AgentAdapter/AgentEvent (PROVIDER_INTEGRATION §P1, packages/agent-core).

```ts
// packages/agent-core/src/agent-adapter.ts
export type AgentBrain = 'codex' | 'claude';

export interface AgentEvent {
  brain: AgentBrain;
  threadId: string;
  turnId?: string;
  ts: number;
  kind:
    | 'thread.started' | 'turn.started' | 'turn.completed' | 'turn.failed'
    | 'message.delta' | 'message' | 'reasoning' | 'command' | 'file_change'
    | 'mcp_tool' | 'web_search' | 'todo' | 'usage' | 'rate_limit' | 'error';
  payload: unknown;              // normalized per kind
  raw?: unknown;                 // original CLI payload for audit
}

export interface ApprovalRequest {
  id: string;                    // JSON-RPC id (number|string) – must echo back
  threadId: string; turnId: string; itemId: string;
  type: 'command' | 'file_change' | 'permissions' | 'user_input';
  command?: string; cwd?: string; reason?: string; grantRoot?: string;
  network?: { host: string; protocol: string };
  availableDecisions: Array<'accept' | 'acceptForSession' | 'decline' | 'cancel'>;
}
export type ApprovalDecision = 'accept' | 'acceptForSession' | 'decline' | 'cancel';

export interface StartThreadOptions {
  cwd: string;
  model?: string;
  reasoningEffort?: 'minimal' | 'low' | 'medium' | 'high' | 'xhigh';
  sandbox: 'read-only' | 'workspace-write' | 'danger-full-access';
  approvalPolicy: 'on-request' | 'never';
  networkAccess?: boolean;
  writableRoots?: string[];
  developerInstructions?: string;      // persona của AI employee
  config?: Record<string, unknown>;    // per-thread config.toml overrides
  ephemeral?: boolean;
}

export interface TurnInput { text: string; images?: string[]; outputSchema?: object }

export interface AgentAdapter {
  readonly brain: AgentBrain;
  health(): Promise<{ ok: boolean; version?: string; auth?: 'chatgpt' | 'apikey' | 'none'; detail?: string }>;
  startThread(opts: StartThreadOptions): Promise<{ threadId: string }>;
  resumeThread(threadId: string, opts?: Partial<StartThreadOptions>): Promise<void>;
  startTurn(threadId: string, input: TurnInput): Promise<{ turnId: string }>;
  interrupt(threadId: string, turnId: string): Promise<void>;
  respondApproval(req: ApprovalRequest, decision: ApprovalDecision): Promise<void>;
  events$: Observable<AgentEvent>;             // rxjs Subject
  approvals$: Observable<ApprovalRequest>;
  rateLimits(): Promise<{ usedPercent: number; resetsAt?: number; window?: number } | null>;
  listThreads(cwd?: string): Promise<Array<{ id: string; preview: string; updatedAt: number }>>;
  dispose(): Promise<void>;
}
```

```ts
// apps/api/src/agents/codex/codex-app-server.client.ts
@Injectable()
export class CodexAppServerClient implements OnModuleDestroy {
  private proc?: ChildProcess;          // spawn(codexBin, ['app-server'], { env: {...process.env, CODEX_HOME} })
  private nextId = 1;
  private pending = new Map<number, { resolve; reject; method: string }>();
  readonly notifications$ = new Subject<{ method: string; params: any }>();
  readonly requests$ = new Subject<{ id: number; method: string; params: any }>();

  async start(): Promise<void> {
    this.proc = spawn(this.bin, ['app-server'], { stdio: ['pipe', 'pipe', 'pipe'], windowsHide: true, env });
    readline.createInterface({ input: this.proc.stdout! }).on('line', (l) => this.onLine(l));
    await this.request('initialize', { clientInfo: { name: 'ai_workspace', title: 'AI Workspace', version: APP_VERSION }, capabilities: { experimentalApi: true } });
    this.notify('initialized', {});
  }
  request<T = any>(method: string, params?: unknown): Promise<T> { /* write {method,id,params}\n ; resolve on {id,result} */ }
  notify(method: string, params?: unknown) { this.proc!.stdin!.write(JSON.stringify({ method, params }) + '\n'); }
  respond(id: number, result: unknown) { this.proc!.stdin!.write(JSON.stringify({ id, result }) + '\n'); }
  private onLine(line: string) {
    const msg = JSON.parse(line);
    if ('id' in msg && ('result' in msg || 'error' in msg)) return this.settle(msg);   // response
    if ('id' in msg && 'method' in msg) return this.requests$.next(msg);               // server->client request (approval...)
    if ('method' in msg) return this.notifications$.next(msg);                        // notification
  }
}
```

```ts
// apps/api/src/agents/codex/codex.adapter.ts
@Injectable()
export class CodexAdapter implements AgentAdapter {
  readonly brain = 'codex' as const;
  constructor(private app: CodexAppServerClient, private execFallback: CodexExecRunner, private cfg: CodexConfig) {}

  async health() {
    // 1) codex --version  2) codex login status (exit 0 => logged in)  3) account/read via app-server
  }
  async startThread(o: StartThreadOptions) {
    const r = await this.app.request('thread/start', {
      cwd: o.cwd, model: o.model, sandbox: o.sandbox, approvalPolicy: o.approvalPolicy,
      developerInstructions: o.developerInstructions, serviceName: 'ai_workspace',
      config: { model_reasoning_effort: o.reasoningEffort, 'sandbox_workspace_write.network_access': o.networkAccess, ...o.config },
      ephemeral: o.ephemeral ?? false,
    });
    return { threadId: r.thread.id };
  }
  async startTurn(threadId: string, input: TurnInput) {
    const r = await this.app.request('turn/start', {
      threadId,
      input: [{ type: 'text', text: input.text }, ...(input.images ?? []).map((p) => ({ type: 'localImage', path: p }))],
      outputSchema: input.outputSchema,
    });
    return { turnId: r.turn.id };
  }
  interrupt = (threadId, turnId) => this.app.request('turn/interrupt', { threadId, turnId });
  respondApproval(req, decision) {
    const id = Number(req.id);
    if (req.type === 'permissions') return this.app.respond(id, { permissions: {}, scope: 'turn' });
    return this.app.respond(id, { decision });
  }
  // mapping: notifications$ -> AgentEvent (camelCase v2 items); requests$ -> approvals$
  // fallback: if (!this.app.alive) this.execFallback.run({ cwd, prompt, json: true, sandbox, config: { approval_policy: 'never' } })
}
```

```ts
// apps/api/src/agents/codex/codex-exec.runner.ts  (fallback / batch)
export class CodexExecRunner {
  run(o: { cwd: string; prompt: string; threadId?: string; sandbox: string; model?: string; outputSchemaFile?: string; lastMessageFile?: string; config?: Record<string, string> }) {
    const args = ['exec', '--json', '--color', 'never', '--skip-git-repo-check', '-C', o.cwd, '--sandbox', o.sandbox];
    for (const [k, v] of Object.entries(o.config ?? {})) args.push('-c', `${k}=${v}`);
    if (o.model) args.push('-m', o.model);
    if (o.outputSchemaFile) args.push('--output-schema', o.outputSchemaFile);
    if (o.lastMessageFile) args.push('-o', o.lastMessageFile);
    if (o.threadId) args.push('resume', o.threadId);
    const child = spawn(bin, args, { windowsHide: true });   // prompt -> stdin, giống SDK
    child.stdin.end(o.prompt);
    // parse stdout JSONL: thread.started | turn.started | item.* (snake_case) | turn.completed{usage} | turn.failed | error
  }
}
```

### 11.3 Luồng vận hành

1. **Boot:** `CodexAdapter.health()` → nếu `codex` không có trong PATH → trạng thái `brain unavailable`, UI hiện hướng dẫn `npm i -g @openai/codex` + `codex login`. Nếu login status ≠ ChatGPT → UI nút "Login" gọi `account/login/start` và mở `authUrl`.
2. **Tạo AI employee:** mỗi employee = 1 Codex thread (persist `threadId`, `cwd`, persona). `thread/start` với `developerInstructions` = persona + rules của công ty.
3. **Giao task:** scheduler kiểm tra `rateLimits()` + concurrency cap → `turn/start`. Stream `item/agentMessage/delta`, `item/completed` lên frontend qua WebSocket gateway.
4. **Approval:** `approvals$` → policy engine (whitelist lệnh an toàn → auto `accept`; còn lại đẩy lên UI manager) → `respondApproval`. Timeout → `decline`.
5. **Kết thúc:** `turn/completed` → lưu usage (`thread/tokenUsage/updated`) vào DB, cập nhật task board, trigger review employee khác.
6. **Crash recovery:** app-server chết → restart, `thread/resume` các thread đang active, `thread/list` để đối chiếu.

### 11.4 Việc cần làm sau khi user cài Codex CLI (live tests pending)

- [ ] `npm i -g @openai/codex` → `codex --version`, `codex login status` (kỳ vọng `Logged in using ChatGPT` vì dùng chung `~/.codex/auth.json`).
- [ ] `codex app-server generate-ts --out packages/codex-protocol/generated` và commit theo version.
- [ ] Đo thực tế: `codex exec --json --skip-git-repo-check -C <tmp> "echo hi"` → ghi lại exit codes, dòng JSONL thật trên Windows.
- [ ] Test app-server stdio: initialize → thread/start → turn/start với `approvalPolicy: "on-request"` + lệnh cần network để thấy `item/commandExecution/requestApproval`.
- [ ] Test sandbox: `codex -c windows.sandbox=mxc sandbox -- cmd /c "echo x > C:\outside.txt"` (kỳ vọng bị chặn) và so với `elevated`.
- [ ] Chạy 3 thread song song, theo dõi `account/rateLimits/updated` để calibrate concurrency cap.

---

## Nguồn

- Help output thực tế từ `codex.exe` 0.164.0-alpha.11.1 (bundled trong VS Code extension) và bindings generate bằng `codex app-server generate-ts --experimental`.
- https://learn.chatgpt.com/docs/app-server (redirect từ developers.openai.com/codex/app-server)
- https://learn.chatgpt.com/docs/non-interactive-mode
- https://learn.chatgpt.com/docs/auth
- https://learn.chatgpt.com/docs/agent-approvals-security.md
- https://learn.chatgpt.com/docs/windows/windows-sandbox.md , https://learn.chatgpt.com/docs/windows/wsl.md
- https://learn.chatgpt.com/docs/config-file/config-reference.md
- https://learn.chatgpt.com/docs/extend/mcp.md
- https://learn.chatgpt.com/docs/codex-sdk và https://github.com/openai/codex/tree/main/sdk/typescript (`src/exec.ts`, `events.ts`, `items.ts`, `threadOptions.ts`, `codexOptions.ts`)
- https://learn.chatgpt.com/docs/pricing (usage limits)
- npm registry: `@openai/codex@0.166.0`, `@openai/codex-sdk@0.166.0`
- Rollout mẫu tại `~/.codex/sessions/2026/10/21/*.jsonl` (chỉ đọc cấu trúc, không ghi lại nội dung).
