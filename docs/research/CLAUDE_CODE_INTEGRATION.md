# Claude Code Integration Research (Agent 05)

> Mục tiêu: xác định cách NestJS backend của AI WORKSPACE điều khiển Claude Code như "bộ não" của từng AI employee,
> tái sử dụng login subscription (OAuth claude.ai) sẵn có của CLI, chạy tới 10 session song song, có approval gating.
>
> Môi trường test thực tế: Windows 11, Claude Code CLI **2.1.312** tại `C:\Users\TUYEN\.local\bin\claude`,
> Agent SDK `@anthropic-ai/claude-agent-sdk@0.3.314`. Ngày test: 2026-10-30.
> Tất cả output bên dưới là output thật (đã redact thông tin nhạy cảm / cắt ngắn mảng dài).
>
> **[Ghi chú 2026-10-30, OWR-2 (Q-17/D-004, Q-07, Q-08, Q-05)]**: tài liệu nghiên cứu, giữ nguyên làm lịch sử. Claude Agent SDK là adapter chính; raw CLI chỉ cho tác vụ phụ trợ/PoC. Phải phân biệt subscription authentication, API authentication và usage limits; SDK không đồng nghĩa được chạy số session không giới hạn. "Tới 10 session song song" là mục tiêu nghiên cứu cũ, không phải xác nhận ToS: mặc định ứng dụng 5, không tự tăng lên 10/18. Không tự động failover provider khi hết quota. Chờ approval tối đa 540 s rồi `WAITING_HUMAN` (giá trị 30 phút của `canUseTool` trong research bị thay thế).

---

## 0. Tóm tắt kết quả (TL;DR)

| Hạng mục | Kết quả |
|---|---|
| `claude -p ... --output-format json` | OK. Trả về 1 object `result` với `session_id`, `total_cost_usd`, `usage`, `modelUsage`, `duration_ms`, `num_turns`, `permission_denials`. |
| `--output-format stream-json --verbose` | OK. NDJSON: `system/init`, `assistant`, `user` (tool_result), `rate_limit_event`, `result`. |
| `--resume <session_id>` | OK. Giữ nguyên `session_id`, nhớ ngữ cảnh. |
| `--input-format stream-json` (multi-turn 1 process) | OK. 1 session, mỗi turn 1 `result` với `result_index` tăng dần. |
| `--json-schema` structured output | OK. Field `structured_output`. |
| Approval gating | `permission_denials` hoạt động; SDK `canUseTool` callback được gọi và chặn được Write. |
| 5 process `claude -p` song song, cùng login | OK, 7s wall-clock, không lỗi rate-limit. |
| Auth | `claude auth status` in JSON: `loggedIn`, `authMethod: "claude.ai"`, `subscriptionType: "team"`. SDK kế thừa OAuth login (`apiKeySource: "none"`, không cần `ANTHROPIC_API_KEY`). |
| Khuyến nghị | **Dùng Agent SDK (`query()`)** làm adapter chính; fallback raw CLI spawn cho healthcheck / trường hợp đặc biệt. |

---

## 1. CLI flags cho programmatic use

### 1.1 `claude --help` (trích các flag liên quan, output thật 2.1.312)

```text
Usage: claude [options] [command] [prompt]

Options:
  --add-dir <directories...>            Additional directories to allow tool access to
  --agent <agent>                       Agent for the current session. Overrides the 'agent' setting.
  --agents <json-or-file>               JSON object defining custom agents, or with --print the path to a file
  --allowedTools, --allowed-tools <tools...>
      Comma or space-separated list of tool names to allow (e.g. "Bash(git *) Edit")
  --append-system-prompt <prompt>       Append a system prompt to the default system prompt
  --bare                                Minimal mode: skip hooks, LSP, plugin sync, auto-memory, keychain reads,
                                        CLAUDE.md auto-discovery. Anthropic auth is strictly ANTHROPIC_API_KEY or
                                        apiKeyHelper via --settings (OAuth and keychain are never read).
  -c, --continue                        Continue the most recent conversation in the current directory
  --dangerously-skip-permissions        Bypass all permission checks. Recommended only for sandboxes
  --disallowedTools, --disallowed-tools <tools...>
      Comma or space-separated list of tool names to deny (e.g. "Bash(git *) Edit")
  --effort <level>                      Effort level (low, medium, high, xhigh, max)
  --fallback-model <model>              Automatic fallback when default model is overloaded
  --fork-session                        When resuming, create a new session ID instead of reusing the original
  --forward-subagent-text               Forward subagent text/thinking as assistant/user messages (stream-json)
  --include-hook-events                 Include all hook lifecycle events in the output stream (stream-json)
  --include-partial-messages            Include partial message chunks (--print + stream-json)
  --input-format <format>               "text" (default) | "stream-json" (realtime streaming input) (--print only)
  --json-schema <schema>                JSON Schema for structured output validation (--print only)
  --max-budget-usd <amount>             Maximum dollar amount to spend on API calls (--print only)
  --mcp-config <configs...>             Load MCP servers from JSON files or strings
  --model <model>                       Model alias ('fable', 'opus', 'sonnet', 'haiku') or full name
  -n, --name <name>                     Display name for this session
  --no-session-persistence              Sessions will not be saved to disk and cannot be resumed (--print only)
  --output-format <format>              "text" | "json" (single result) | "stream-json" (realtime) (--print only)
  --permission-mode <mode>              "acceptEdits" | "auto" | "bypassPermissions" | "manual" | "dontAsk" | "plan"
  --permission-prompts <target>         "host" (SDK host or --permission-prompt-tool) | "none" (deny anything
                                        that would prompt)  (default: "host")
  -p, --print                           Print response and exit. Workspace trust dialog is skipped.
  --replay-user-messages                Re-emit user messages from stdin back on stdout (stream-json in+out)
  --restricted                          Removes Bash/PowerShell/WebFetch unless --tools names them; confines file
                                        tools to working dirs; refuses bypassPermissions
  -r, --resume [value]                  Resume a conversation by session ID or name
  --session-id <uuid>                   Use a specific session ID (must be a valid UUID)
  --setting-sources <sources>           Comma-separated: user, project, local
  --settings <file-or-json>             Settings JSON file or string
  --strict-mcp-config                   Only use MCP servers from --mcp-config
  --system-prompt <prompt>              System prompt to use for the session (replace default)
  --tools <tools...>                    Built-in tool set: "" (none), "default", or "Bash,Edit,Read"
  --verbose                             Override verbose mode setting from config
  -w, --worktree [name]                 Create a new git worktree for this session

Commands:
  auth                                  Manage authentication  (login | logout | status)
  mcp                                   Configure and manage MCP servers
  agents / attach / logs / stop / rm    Background sessions (claude --bg)
  setup-token                           Long-lived auth token (requires Claude subscription)
```

Ghi chú quan trọng:

- `--permission-prompt-tool` **không còn xuất hiện** trong `--help` 2.1.312 nhưng vẫn được docs và `--permission-prompts host` nhắc tới; SDK có `permissionPromptToolName`. Trong SDK nên dùng `canUseTool` thay thế.
- Không có `claude -p --help` riêng; `-p` chỉ là flag.
- `--bare`: nhanh hơn nhưng **không đọc OAuth** => **không dùng được với subscription login**. Với AI WORKSPACE (muốn tái dùng login) => KHÔNG dùng `--bare`; thay vào đó dùng `--setting-sources user` + `--strict-mcp-config` + `--settings '{"disableAllHooks":true}'` để giảm nhiễu.
- Từ v2.1.302, `claude -p` khởi động ở **Manual mode** nếu không set permission mode; mọi tool cần duyệt sẽ bị deny trừ khi có `--allowedTools`, `PermissionRequest` hook, `--permission-prompt-tool` hoặc SDK `canUseTool`.
- `-p` không có workspace trust dialog nhưng **vẫn load** `.claude/settings.json` + `.mcp.json` của repo.

### 1.2 Permission modes (docs `code.claude.com/docs/en/permissions`)

| Mode | Ý nghĩa |
|---|---|
| `default` (Manual) | Hỏi với mọi hành động cần duyệt (Bash không read-only, Edit/Write, WebFetch...). Trong `-p` không có host => deny. |
| `acceptEdits` | Tự chấp nhận edit file + `mkdir/touch/mv/cp` trong working dir / `additionalDirectories`. |
| `plan` | Chỉ read-only. |
| `dontAsk` | Deny mọi thứ lẽ ra phải hỏi; chỉ chạy những gì đã allow (`--allowedTools`, `permissions.allow`). |
| `auto` | Classifier tự duyệt phần lớn hành động. |
| `bypassPermissions` | Bỏ qua mọi prompt (kể cả ghi `.git`, `.claude`). Có thể bị khóa bằng `permissions.disableBypassPermissionsMode`. |

Precedence: deny rule > allow rule; hook exit 2 chặn trước khi evaluate rule; `--permission-prompt-tool`/`canUseTool` chỉ được hỏi khi flow "rơi xuống prompt".

---

## 2. Auth detection và cấu trúc `~/.claude`

### 2.1 `claude auth status` (JSON mặc định, không cần `--json`)

```text
$ claude auth --help
Usage: claude auth [options] [command]
Commands:
  login [options]   Sign in to your Anthropic account
  logout            Log out from your Anthropic account
  status [options]  Show authentication status

$ claude auth status
{
  "loggedIn": true,
  "authMethod": "claude.ai",
  "apiProvider": "firstParty",
  "analyticsDisabled": false,
  "projectsDirectory": "C:\\Users\\TUYEN\\.claude\\projects",
  "configDirectory": "C:\\Users\\TUYEN\\.claude",
  "email": "<redacted>",
  "orgId": "<redacted-uuid>",
  "orgName": "<redacted>",
  "subscriptionType": "team"
}
```

Cách detect an toàn trong NestJS (không đụng token):

```ts
// execFile('claude', ['auth', 'status']) -> parse JSON
type AuthStatus = {
  loggedIn: boolean;
  authMethod: 'claude.ai' | 'console' | 'apiKey' | string;   // "claude.ai" = OAuth subscription
  apiProvider: 'firstParty' | 'bedrock' | 'vertex' | 'foundry' | 'gateway';
  subscriptionType?: 'pro' | 'max' | 'team' | 'enterprise' | string;
  email?: string; orgId?: string; orgName?: string;
};
const isSubscription = s.loggedIn && s.authMethod === 'claude.ai' && s.apiProvider === 'firstParty';
```

Ngoài ra `system/init` event của mỗi session có `apiKeySource`: `'ANTHROPIC_API_KEY' | 'apiKeyHelper' | '/login managed key' | 'none'`.
`'none'` = đang dùng OAuth claude.ai (xác nhận trong smoke test SDK bên dưới). SDK còn có `query.accountInfo()` trả `{ email, organization, subscriptionType, tokenSource, apiKeySource, apiProvider }`.

### 2.2 Cấu trúc `~/.claude/` (chỉ list, KHÔNG đọc credentials)

```text
~/.claude/
  .credentials.json          <- OAuth tokens (file người dùng đọc được bởi tiến trình cùng user, kiểm soát bằng ACL). KHÔNG BAO GIỜ đọc/log/copy. CLI tự refresh.
  settings.json              <- user settings (hooks, permissions, model...)
  history.jsonl              <- prompt history
  projects/<slug>/           <- slug = cwd với ':' '\' '/' -> '-'  (vd: d--Code-Tuyen-Agent-management)
      <session-uuid>.jsonl   <- transcript (append-only NDJSON)
      <session-uuid>/        <- subagents/, tool-results/ của session
      memory/                <- auto-memory
  sessions/  session-env/  shell-snapshots/  file-history/  backups/
  hooks/  skills/  plugins/  ide/  state/  telemetry/  cache/  daemon/
  policy-limits.json(+signature)  remote-settings.json(+signature)
```

Trên Windows, credentials nằm ở file `.credentials.json` (bản này không thấy dùng Windows Credential Manager; trên macOS là Keychain). Policy: adapter chỉ gọi `claude`, không bao giờ truy cập file này.

### 2.3 Transcript JSONL format (sample redacted từ `projects/D--Code-Tuyen-OrderFlow/<uuid>.jsonl`)

```json
{"type":"queue-operation","operation":"enqueue","timestamp":"2026-10-29T15:31:57.378Z","sessionId":"a1b73c4d-..."}
{"parentUuid":null,"isSidechain":false,"promptId":"934d1551-...","type":"user",
 "message":{"role":"user","content":[{"type":"text","text":"<redacted>"}]},
 "uuid":"c99a7408-...","timestamp":"2026-10-29T15:31:57.596Z","permissionMode":"auto",
 "origin":{"kind":"human"},"promptSource":"sdk","turnOrigin":"human","turnPosition":{"promptIndex":1,"turnIndex":1}}
{"parentUuid":"c99a7408-...","type":"attachment","attachment":{"type":"environment","snapshot":{"workingDirectory":"<redacted>","isGitRepo":true,"platform":"win32"}}}
{"parentUuid":"...","type":"attachment","attachment":{"type":"model","identity":{"modelId":"claude-sonnet-5-5","marketingName":"Sonnet 5.5"}}}
{"parentUuid":"...","type":"assistant","message":{"role":"assistant","content":[{"type":"tool_use","name":"Read","id":"toolu_..."}]},"uuid":"...","timestamp":"..."}
```

Mỗi dòng có `type` (`user` | `assistant` | `attachment` | `queue-operation` | `summary` ...), `uuid`/`parentUuid` tạo thành cây, `sessionId`, `timestamp`, `cwd`, `gitBranch`. Pixel Agents dùng chính file này (poll + fs.watch) để quan sát agent; AI WORKSPACE không cần vì nhận stream trực tiếp từ stdout/SDK, nhưng có thể dùng làm **audit log thứ cấp** và để `--resume <path-to-jsonl>`.

### 2.4 Cách Pixel Agents tích hợp (tham khảo, `pixel-agents/server/src/providers/hook/claude/claude.ts`)

```ts
function buildLaunchCommand(sessionId, cwd, opts) {
  const args = ['--session-id', sessionId];
  if (opts?.bypassPermissions) args.push('--dangerously-skip-permissions');
  return { command: 'claude', args, env: { PWD: cwd } };
}
```

- Sinh UUID trước, chạy `claude --session-id <uuid>` **tương tác** trong VS Code terminal, rồi chờ file `~/.claude/projects/<slug>/<uuid>.jsonl` xuất hiện và poll nó.
- Cài hook script `~/.pixel-agents/hooks/claude-hook.js` vào `~/.claude/settings.json` cho các event `SessionStart, UserPromptSubmit, PermissionRequest, Notification, PreToolUse, PostToolUse, PostToolUseFailure, SubagentStart, SubagentStop, TeammateIdle, TaskCompleted, Stop, SessionEnd`; script POST payload tới `http://127.0.0.1:<port>/api/hooks/claude` với Bearer token đọc từ `~/.pixel-agents/server.json`.
- Phù hợp cho "quan sát session người dùng tự chạy"; **không phù hợp** cho backend điều khiển headless (không có input/approval programmatic).

---

## 3. Smoke tests (output thật)

### 3.1 `--output-format json`

```text
$ cd <scratch>/smoke1 && claude -p "Reply with exactly: PONG" --output-format json --max-turns 1
{"type":"result","subtype":"success","is_error":false,"result":"PONG",
 "session_id":"d79c6dbb-6cee-4e0f-9863-b3c3fd74ce41","num_turns":1,
 "duration_ms":1720,"duration_api_ms":2280,"ttft_ms":1690,"stop_reason":"end_turn",
 "total_cost_usd":0.375653,
 "usage":{"input_tokens":2,"cache_creation_input_tokens":18388,"cache_read_input_tokens":29992,"output_tokens":5,
          "service_tier":"standard","cache_creation":{"ephemeral_1h_input_tokens":18388,"ephemeral_5m_input_tokens":0}},
 "modelUsage":{"claude-haiku-5-5":{"inputTokens":1175,"outputTokens":15,"costUSD":0.000125,"costBasis":"list","provider":"firstParty"},
               "claude-fable-5-1":{"inputTokens":2,"outputTokens":5,"cacheReadInputTokens":29992,"cacheCreationInputTokens":18388,
                                   "costUSD":0.375528,"contextWindow":1000000,"maxOutputTokens":64000,"costBasis":"list"}},
 "permission_denials":[],"terminal_reason":"completed","api_error_status":null,"safety_stops":0,
 "subagent_stats":{"spawned":0,...},"uuid":"3203d056-...","result_index":0}
real 0m6.091s   EXIT=0
```

Nhận xét:
- `total_cost_usd` là **ước tính client-side theo list price** (`costBasis: "list"`); với subscription không bị bill. Vẫn hữu ích để đo "mức tiêu thụ" tương đối giữa các agent.
- Model mặc định lấy từ user settings (`fable`). Phải truyền `--model` tường minh cho từng agent để kiểm soát chi phí/quota.
- Startup ~4s (load settings, MCP claude.ai connectors, hooks). Với 10 agent cần giảm bằng `--strict-mcp-config`, `--setting-sources user`, `--settings '{"disableAllHooks":true}'`.

### 3.2 `--output-format stream-json --verbose` (NDJSON schema)

```text
$ claude -p "Reply with exactly: PONG" --output-format stream-json --verbose --max-turns 1 --model haiku
{"type":"system","subtype":"hook_started","hook_id":"...","hook_name":"SessionStart:startup","hook_event":"SessionStart","session_id":"3171fb53-..."}
{"type":"system","subtype":"hook_response","hook_id":"...","exit_code":0,"outcome":"success","session_id":"3171fb53-..."}
{"type":"system","subtype":"init","cwd":"...\\smoke2","session_id":"3171fb53-d4f7-4729-b177-aa75923d3a65",
 "tools":["Task","Artifact",... 41 items],"mcp_servers":[{"name":"claude.ai Claude Docs","status":"connected","source":"claudeai"},...],
 "model":"claude-haiku-5-5","permissionMode":"default","apiKeySource":"none","claude_code_version":"2.1.312",
 "output_style":"default","agents":[...],"skills":[...],"plugins":[...],"capabilities":[...],
 "scratchpad_path":"...","original_cwd":"...","additional_directories":[],"uuid":"..."}
{"type":"system","subtype":"thinking_tokens","estimated_tokens":50,"estimated_tokens_delta":50,"session_id":"..."}
{"type":"assistant","message":{"model":"claude-haiku-5-5","id":"msg_011C...","role":"assistant",
   "content":[{"type":"thinking","thinking":"","signature":"..."}],"usage":{...}},"parent_tool_use_id":null,"session_id":"...","uuid":"...","timestamp":"2026-10-30T02:47:05.838Z","request_id":"req_..."}
{"type":"assistant","message":{...,"content":[{"type":"text","text":"PONG"}],"stop_reason":null,"usage":{"input_tokens":2,"cache_read_input_tokens":29992,"output_tokens":5}},"parent_tool_use_id":null,"session_id":"..."}
{"type":"rate_limit_event","rate_limit_info":{"status":"allowed","resetsAt":1793342400,"rateLimitType":"five_hour",
   "overageStatus":"rejected","overageDisabledReason":"org_level_disabled","isUsingOverage":false,
   "unifiedWindows":{"five_hour":{"utilization":0.15,"resetsAt":1793342400,"status":"allowed"},
                     "seven_day":{"utilization":0.74,"resetsAt":1793329200,"status":"allowed"}}},"session_id":"..."}
{"type":"result","subtype":"success","is_error":false,"result":"PONG","session_id":"3171fb53-...","num_turns":1,
 "total_cost_usd":0.00378652,"duration_ms":...,"usage":{...},"modelUsage":{...},"permission_denials":[]}
EXIT=0
```

Các event type gặp thực tế: `system` (subtype `init`, `hook_started`, `hook_response`, `thinking_tokens`, `api_retry`, `permission_denied`), `assistant`, `user`, `rate_limit_event`, `result`. Với `--include-partial-messages` thêm `stream_event` (text_delta). `rate_limit_event` cho biết **utilization của cửa sổ 5h và 7 ngày** của subscription — dùng để throttle scheduler.

### 3.3 Tool use: `assistant.tool_use` → `user.tool_result`

```text
$ claude -p "Use the Read tool to read note.txt ..." --output-format stream-json --verbose --max-turns 3 --model haiku \
    --allowedTools Read --permission-mode dontAsk --permission-prompts none
{"type":"system","subtype":"init","permissionMode":"dontAsk","model":"claude-haiku-5-5","apiKeySource":"none",...}
{"type":"assistant","content":[{"type":"tool_use","name":"Read","id":"toolu_01JddS8yAtDdkCJMN1spKraB","input":{"file_path":"...\\smoke3\\note.txt"}}],"parent_tool_use_id":null}
{"type":"user","content":[{"type":"tool_result","tool_use_id":"toolu_01JddS8yAtDdkCJMN1spKraB","content":"1\thello from file\n2\t"}],
 "parent_tool_use_id":null,"tool_use_result":{"type":"text","file":{...}}}
{"type":"assistant","content":[{"type":"text","text":"hello from file"}]}
{"type":"result","subtype":"success","result":"hello from file","num_turns":2,"permission_denials":[],"total_cost_usd":0.00457915}
```

`parent_tool_use_id` = null cho main conversation; subagent messages mang id của `Agent` tool_use (dùng `--forward-subagent-text` để lấy cả text/thinking của subagent).

### 3.4 `--resume <session_id>`

```text
$ claude -p "What exact word did you reply with in your previous message? Answer with only that word." \
    --resume d79c6dbb-6cee-4e0f-9863-b3c3fd74ce41 --output-format json --max-turns 1 --model haiku
{"type":"result","subtype":"success","is_error":false,"result":"PONG",
 "session_id":"d79c6dbb-6cee-4e0f-9863-b3c3fd74ce41","num_turns":1,"total_cost_usd":0.37977602,"duration_ms":1748}
```

- Session ID giữ nguyên; `total_cost_usd` là **tổng tích lũy** cả session (docs: "the run reports the conversation's whole total").
- Resume được từ bất kỳ cwd nào (tìm theo ID trong mọi project); có thể truyền path `.jsonl` thay ID. `--fork-session` để tạo nhánh mới.
- Có thể dùng `--session-id <uuid>` (UUID do ta sinh) để biết trước ID — cách Pixel Agents dùng.

### 3.5 `--input-format stream-json` (multi-turn trong 1 process)

```text
$ cat in.ndjson
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"Remember the secret word is MANGO. Reply only: OK"}]}}
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"What is the secret word? Reply with only the word."}]}}
$ cat in.ndjson | claude -p --input-format stream-json --output-format stream-json --verbose --model haiku --replay-user-messages
{"type":"user","content":[{"type":"text","text":"Remember the secret word is MANGO. Reply only: OK"}],"session_id":"1a5ac42f-..."}
{"type":"assistant","content":[{"type":"text","text":"OK"}],"session_id":"1a5ac42f-..."}
{"type":"result","subtype":"success","result":"OK","num_turns":1,"session_id":"1a5ac42f-...","result_index":0}
{"type":"user","content":[{"type":"text","text":"What is the secret word? ..."}],"session_id":"1a5ac42f-..."}
{"type":"assistant","content":[{"type":"text","text":"MANGO"}],"session_id":"1a5ac42f-..."}
{"type":"result","subtype":"success","result":"MANGO","num_turns":1,"session_id":"1a5ac42f-...","result_index":1}
```

=> Một process = một agent "sống lâu": giữ stdin mở, ghi từng `{"type":"user",...}` khi có task mới, đọc `result` với `result_index`. Đây chính là cơ chế SDK `query({ prompt: AsyncIterable })` + `streamInput()` dùng bên dưới.

### 3.6 `--json-schema` structured output

```text
$ claude -p "Give the name Alice and age 30" --output-format json --model haiku \
   --json-schema '{"type":"object","properties":{"name":{"type":"string"},"age":{"type":"number"}},"required":["name","age"]}'
{"subtype":"success","result":"{\"name\":\"Alice\",\"age\":30}","structured_output":{"name":"Alice","age":30},"is_error":false}
```

### 3.7 Permission denial (Manual mode, không có host)

```text
$ claude -p "Create a file named out.txt ... using the Write tool. If denied, reply DENIED." \
    --output-format json --model haiku --max-turns 2 --permission-mode manual --permission-prompts none
{"subtype":"success","result":"DENIED\n\nThe Write call for `out.txt` was refused. The session has no approval prompt, ...",
 "permission_denials":[{"tool_name":"Write","tool_use_id":"toolu_01SnsNvy9kQ6DR3YdART6Vva",
                        "tool_input":{"file_path":"...\\smoke6\\out.txt","content":"hello"}}],"is_error":false,"num_turns":2}
```

`--disallowedTools Bash` thì tool bị **gỡ khỏi context** (model không thấy Bash) => `permission_denials: []`.

### 3.8 Song song 5 process cùng login

```text
$ for i in 1..5: ( cd pp$i && claude -p "Reply with exactly: PONG$i" --output-format json --model haiku --max-turns 1 ) &
WALL=7s
pp1 PONG1 session 98a9032b-...  duration_ms 1118  is_error false  api_error_status null
pp2 PONG2 session 815a8fde-...  duration_ms 2219
pp3 PONG3 session c7a2a0ce-...  duration_ms 1054
pp4 PONG4 session 34eeda54-...  duration_ms 1102
pp5 PONG5 session c85757f5-...  duration_ms 1029
```

Không có lock file / lỗi "already running". Mỗi process có session riêng, cùng `.credentials.json` (CLI tự refresh token; không quan sát lỗi khi nhiều process chạy đồng thời).

---

## 4. TypeScript Agent SDK `@anthropic-ai/claude-agent-sdk`

### 4.1 Package

```text
$ npm view @anthropic-ai/claude-agent-sdk version  -> 0.3.314
peerDependencies: zod ^4, @anthropic-ai/sdk >=0.93.0, @modelcontextprotocol/sdk ^1.29.0 ; engines node >=18.14
optionalDependencies: @anthropic-ai/claude-agent-sdk-win32-x64 (chứa claude.exe), -linux-x64, -darwin-arm64, ...
```

SDK **bundle sẵn native binary Claude Code** cùng version ("SDK v0.3.191 bundles Claude Code v2.1.191"; "Most installs need no separate Claude Code install"). Có thể trỏ `pathToClaudeCodeExecutable` tới `C:\Users\TUYEN\.local\bin\claude` nếu muốn dùng đúng bản đã cài. SDK spawn subprocess và nói chuyện qua stdin/stdout stream-json + control protocol (`CLAUDE_CODE_ENTRYPOINT=sdk-ts`).

### 4.2 `query()` API (từ `sdk.d.ts` 0.3.314)

```ts
export declare function query(_params: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;

export declare interface Query extends AsyncGenerator<SDKMessage, void> {
  interrupt(): Promise<SDKControlInterruptResponse | undefined>;   // chỉ streaming-input mode
  setPermissionMode(mode: PermissionMode): Promise<void>;
  setModel(model?: string): Promise<void>;
  streamInput(stream: AsyncIterable<SDKUserMessage>): Promise<void>;
  accountInfo(): Promise<AccountInfo>;
  getContextUsage(): Promise<...>;  mcpServerStatus(); setMcpServers(); supportedModels(); rewindFiles(...);
  stopTask(taskId: string): Promise<void>;
  close(): void;   // "terminate the underlying process"
}

export declare type PermissionMode = 'default' | 'acceptEdits' | 'bypassPermissions' | 'plan' | 'dontAsk' | 'auto';

export declare type CanUseTool = (toolName: string, input: Record<string, unknown>, options: {
  signal: AbortSignal; suggestions?: PermissionUpdate[]; blockedPath?: string;
  toolUseID: string; agentID?: string; description?: string; ...
}) => Promise<PermissionResult>;

export declare type PermissionResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown>; updatedPermissions?: PermissionUpdate[]; toolUseID?: string }
  | { behavior: 'deny'; message: string; interrupt?: boolean; toolUseID?: string };

export declare type HookCallback = (input: HookInput, toolUseID: string | undefined, options: { signal: AbortSignal })
  => Promise<HookJSONOutput>;
```

Options quan trọng (tên thật trong `Options` type):

| Option | Ghi chú |
|---|---|
| `cwd`, `additionalDirectories` | Working dir của agent (tương đương `--add-dir`). |
| `model`, `fallbackModel`, `effort`, `thinking`, `maxThinkingTokens` | Chọn model/effort per agent. |
| `allowedTools`, `disallowedTools`, `tools` | Rule syntax `Bash(git *)`. |
| `permissionMode`, `allowDangerouslySkipPermissions`, `permissionPrompts: 'host' \| 'none'`, `permissionPromptToolName` | Gating. |
| `canUseTool` | Callback duyệt tool khi flow rơi xuống prompt (**cơ chế approval chính**). |
| `hooks: Partial<Record<HookEvent, HookCallbackMatcher[]>>` | Hook in-process (không cần settings.json). |
| `mcpServers`, `strictMcpConfig` | MCP per agent; `createSdkMcpServer()` tạo in-process MCP tool. |
| `resume`, `continue`, `forkSession`, `sessionId`, `resumeSessionAt`, `persistSession` | Session lifecycle. |
| `maxTurns`, `maxBudgetUsd`, `abortController` | Giới hạn + cancel. |
| `systemPrompt: string \| { type:'preset', preset:'claude_code', append?: string }` | `append` = `--append-system-prompt`. |
| `settings`, `settingSources: ('user'\|'project'\|'local')[]`, `managedSettings` | `[]` = bỏ qua mọi settings file. |
| `env` | **Thay thế** (không merge) env của subprocess. Nếu set phải copy `process.env` vào (HOME/USERPROFILE cần cho OAuth). |
| `includePartialMessages`, `includeHookEvents`, `forwardSubagentText` | Streaming chi tiết. |
| `outputFormat: { type:'json_schema', schema }` | Structured output. |
| `pathToClaudeCodeExecutable`, `executable`, `spawnClaudeCodeProcess` | Tuỳ biến spawn (vd chạy trong VM/container). |
| `stderr: (data) => void`, `debug`, `debugFile` | Log. |

`SDKMessage` union (rút gọn): `SDKAssistantMessage | SDKUserMessage | SDKUserMessageReplay | SDKResultMessage (Success|Error) | SDKSystemMessage | SDKPartialAssistantMessage | SDKCompactBoundaryMessage | SDKStatusMessage | SDKAPIRetryMessage | SDKHookStartedMessage | SDKHookResponseMessage | SDKToolProgressMessage | SDKAuthStatusMessage | SDKTaskNotificationMessage | SDKThinkingTokensMessage | SDKRateLimitEvent | SDKPermissionDeniedMessage | SDKInformationalMessage | ...`.

`HookEvent` = `'PreToolUse' | 'PostToolUse' | 'PostToolUseFailure' | 'PostToolBatch' | 'Notification' | 'UserPromptSubmit' | 'SessionStart' | 'SessionEnd' | 'Stop' | 'StopFailure' | 'SubagentStart' | 'SubagentStop' | 'PreCompact' | 'PostCompact' | 'PermissionRequest' | 'PermissionDenied' | 'Setup' | 'TeammateIdle' | 'TaskCreated' | 'TaskCompleted' | 'Elicitation' | 'ConfigChange' | 'WorktreeCreate' | 'WorktreeRemove' | 'CwdChanged' | 'FileChanged' | ...`.

### 4.3 SDK có dùng OAuth login của CLI không? — ĐÃ XÁC MINH: CÓ

Smoke test thật (`env -u ANTHROPIC_API_KEY node probe.mjs`, SDK dùng binary bundled `claude-agent-sdk-win32-x64/claude.exe`):

```text
INIT apiKeySource= none model= claude-haiku-5-5 mode= default session= 8667881f-9dca-4723-840a-0943ccd31b8a
ASSISTANT ["PONG"]
ASSISTANT [{"tool":"Write"}]
HOOK PreToolUse Write                                   <- SDK in-process hook chạy
USER tool_result [{"type":"tool_result","content":"Human approval required (demo)","is_error":true,"tool_use_id":"toolu_01Wdx9..."}]
ASSISTANT ["DENIED"]
RESULT {"subtype":"success","result":"DENIED","num_turns":2,"cost":0.01012404,"denials":1}
canUseTool calls: [{"toolName":"Write","input":{"file_path":"...\\sdkprobe\\out.txt","content":"hello"}}]
EXIT=0    ANTHROPIC_API_KEY set? no
```

- `apiKeySource: "none"` + không có `ANTHROPIC_API_KEY` + chạy thành công => subprocess đọc `~/.claude/.credentials.json` (OAuth claude.ai) y hệt CLI tương tác. `sdk.d.ts` cũng ghi: *"Anthropic OAuth login only applies when apiProvider = firstParty"*.
- `canUseTool` được gọi với `toolName`, `input`; trả `deny` => tool_result `is_error: true` và `permission_denials` = 1.
- Lưu ý: nếu set `CLAUDE_CODE_OAUTH_TOKEN` (từ `claude setup-token`) thì session chạy bằng token đó thay vì login hiện tại. Không cần cho AI WORKSPACE.

### 4.4 So sánh SDK vs raw CLI spawn

| Tiêu chí | Agent SDK `query()` | Raw `spawn('claude', ['-p', ...])` |
|---|---|---|
| Auth | Kế thừa OAuth (đã verify) | Kế thừa OAuth (đã verify) |
| Approval gating | `canUseTool` callback in-process, có `updatedInput`, `interrupt` | Phải tự dựng MCP server cho `--permission-prompt-tool`, hoặc HTTP `PermissionRequest` hook |
| Hooks | In-process callback, typed | Phải ghi settings.json / `--settings` JSON với `type: "http"` |
| Streaming/types | `SDKMessage` typed union, `includePartialMessages` | Tự parse NDJSON, tự định nghĩa types |
| Multi-turn | `streamInput()`, `interrupt()`, `setModel()`, `setPermissionMode()` runtime | `--input-format stream-json` + viết stdin; không có interrupt chuẩn ngoài SIGINT |
| Cancel | `abortController.abort()` / `close()` (SDK đóng stdin trước, grace ~2s rồi kill) | SIGTERM => exit 143, turn dang dở không ghi result |
| Version | Binary bundled theo SDK version (deterministic, không phụ thuộc user update CLI) | Phụ thuộc `claude` trên PATH (user tự update, có thể lệch) |
| Chi phí vận hành | 1 dependency npm (+ binary lớn/platform) | Zero dependency |
| Điểm yếu | API bề mặt rất lớn, thay đổi nhanh (0.3.x); `env` thay thế toàn bộ env | Nhiều glue code; approval phải qua MCP/HTTP |

**Kết luận: dùng Agent SDK.** Giữ một `RawCliClaudeAdapter` nhỏ (chỉ `-p --output-format json`) làm fallback/healthcheck (`claude auth status`, `claude --version`).

---

## 5. Hooks và cách nối vào event system

### 5.1 Cấu hình hook (settings.json / `--settings` JSON / SDK `hooks`)

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash|Write|Edit",
        "hooks": [ { "type": "http", "url": "http://127.0.0.1:3100/api/hooks/claude",
                     "headers": { "Authorization": "Bearer $AIW_HOOK_TOKEN" },
                     "allowedEnvVars": ["AIW_HOOK_TOKEN"], "timeout": 30 } ] }
    ],
    "PostToolUse": [ { "matcher": "*", "hooks": [ { "type": "command", "command": "node ~/.aiw/hooks/claude-hook.js" } ] } ],
    "Stop": [], "Notification": [], "SessionStart": [], "SessionEnd": [], "PermissionRequest": []
  }
}
```

- Types: `command` (stdin JSON, exit code), `http` (POST JSON body = cùng payload), `mcp_tool`, `prompt`, `agent`.
- Payload chung: `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`, `prompt_id`; tool events thêm `tool_name`, `tool_input`, `tool_use_id`.
- Exit code: `0` = ok (stdout JSON được parse), `2` = **block** (stderr = lý do; PreToolUse chặn tool), khác = non-blocking error.
- JSON output: `{"continue":false,"stopReason":"..."}`, `{"decision":"block","reason":"..."}`,
  PreToolUse: `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow|deny|ask|defer","permissionDecisionReason":"...","updatedInput":{...}}}`,
  PermissionRequest: `{"hookSpecificOutput":{"hookEventName":"PermissionRequest","decision":{"behavior":"allow","updatedInput":{...}}}}`.
- Hooks cùng event chạy **song song**; timeout mặc định 600s (command/http); `SessionEnd` chỉ có 1.5s.
- Matcher: `"*"`/`""`/bỏ trống = tất cả; `Edit|Write` = exact list; ký tự khác => regex.

### 5.2 Khuyến nghị cho AI WORKSPACE

- **Với SDK**: dùng `options.hooks` (in-process) => không phải sửa `~/.claude/settings.json` của user, không rò rỉ sang session tương tác của họ. Mỗi hook callback push event vào `EventBus` NestJS (`agent.tool.started`, `agent.tool.finished`, `agent.stopped`, `agent.notification`).
- Dùng `includeHookEvents: true` nếu muốn thấy cả hook của user trong stream.
- Nếu sau này cần quan sát session **không do ta spawn** (user chạy `claude` tay trong worktree) => thêm HTTP hook kiểu Pixel Agents, ghi vào **project** `.claude/settings.json` của worktree (không đụng user settings).
- `settings: { disableAllHooks: true }` (hoặc `settingSources: []`) để tắt hook/MCP lạ của user khi chạy agent production.

---

## 6. Concurrency

- **Đã verify**: 5 process song song, cùng OAuth login, không lỗi. Không có lock toàn cục; mỗi process có `session_id` + transcript riêng.
- **Giới hạn thật sự = quota subscription**, không phải số process: `rate_limit_event.unifiedWindows.five_hour.utilization` / `seven_day.utilization` (test: 0.15 / 0.74, status `allowed_warning` khi vượt ngưỡng 0.75). Team plan: "per-seat allowance that resets on a rolling five-hour window and a weekly window", chia sẻ với Claude chat. Khi hết: lỗi "You've hit your session limit"/"weekly limit", `system/api_retry` với `error: "rate_limit"`, `overageStatus: "rejected"` (org đã tắt overage).
- Docs: "Agent teams use approximately 7x more tokens"; mỗi instance có context window riêng => 10 agent ≈ 10 lần token. Dùng `haiku`/`sonnet` cho worker, `opus`/`fable` cho lead; `effort: low|medium` cho task đơn giản; `maxBudgetUsd` + `maxTurns` per task.
- Scheduler nên: (1) đọc `rate_limit_event` từ mọi session, giữ `maxUtilization` chung; (2) pause queue khi `seven_day.utilization > 0.9` hoặc `status != allowed`; (3) retry sau `resetsAt`; (4) giới hạn `MAX_PARALLEL_AGENTS` (mặc định 10 trong nghiên cứu; thay bằng mặc định 5, trần 18 theo D-010 APPROVED) + semaphore.
- **Isolation cwd**: mỗi agent 1 `git worktree` (`git worktree add ../wt/<agent>-<task> -b agent/<task>`), truyền `cwd` + `additionalDirectories` (chỉ các dir được phép). CLI có `--worktree` nhưng ta nên tự quản lý để kiểm soát lifecycle/cleanup. Transcript được lưu theo slug của cwd => mỗi worktree là 1 "project" riêng trong `~/.claude/projects/`.
- Startup cost ~3–5s/process (settings + claude.ai MCP connectors). Giảm bằng `strictMcpConfig`, `settingSources: ['user']`, tắt plugin, hoặc giữ process sống lâu (streaming input) thay vì spawn mỗi task.
- Windows: path dài, `\\` trong JSON; `spawn` với `shell: false`; `claude` là shim trong `~/.local/bin` — SDK bundled binary tránh vấn đề PATH. Lưu ý Bash heredoc/argv rất dài trên Windows có thể lỗi `ENAMETOOLONG` => truyền prompt dài qua stdin (stream-json) chứ không qua argv.

---

## 7. Permission / approval gating — thiết kế

Ba cơ chế khả thi:

| Cơ chế | Cách hoạt động | Nhận xét |
|---|---|---|
| **A. SDK `canUseTool`** | CLI gửi control request → callback Promise trong NestJS → ta tạo `ApprovalRequest` (DB + WebSocket tới UI) → resolve `{behavior:'allow'\|'deny', updatedInput?}` | Typed, in-process, có `signal` để timeout, có `updatedInput` để sửa lệnh. **Đã test OK.** |
| B. `--permission-prompt-tool mcp__aiw__approve` | Ta chạy MCP server (stdio) expose tool nhận `{tool_name, input, tool_use_id}` và trả JSON `{"behavior":"allow","updatedInput":{...}}` hoặc `{"behavior":"deny","message":"..."}` | Cần thêm process MCP per agent; chỉ hữu ích cho raw CLI. |
| C. Hook `PreToolUse`/`PermissionRequest` (http) exit 2 / `permissionDecision` | Hook POST tới backend; backend trả `deny`/`allow`/`ask` | Hook có timeout (600s) nên có thể "chờ người"; nhưng là sync HTTP, khó scale, không typed. Tốt cho **policy tự động** (deny `rm -rf`, deny ngoài worktree). |

**Khuyến nghị: A làm approval chính + C (SDK in-process `PreToolUse` hook) cho policy cứng.**

Luồng: `permissionMode: 'default'` (Manual) + `allowedTools` whitelist an toàn (`Read`, `Grep`, `Glob`, `Bash(git status *)`, `Bash(npm test *)`, `Edit`, `Write` trong worktree nếu role cho phép) → mọi thứ khác rơi vào `canUseTool` → UI "Approve / Deny / Edit & Approve" → timeout (vd 30 phút) => deny với message "no approver". Trường hợp "autonomous mode" của role: `permissionMode: 'acceptEdits'` hoặc `'auto'` + vẫn `canUseTool` cho Bash nguy hiểm. **Không dùng** `bypassPermissions` ngoài sandbox.

```ts
canUseTool: async (toolName, input, { signal, toolUseID }) => {
  if (policy.autoAllow(agent.role, toolName, input)) return { behavior: 'allow' };
  const req = await approvals.create({ agentId, sessionId, toolName, input, toolUseID });
  events.emit('approval.requested', req);
  const decision = await approvals.waitFor(req.id, { signal, timeoutMs: 30 * 60_000 });
  return decision.approved
    ? { behavior: 'allow', updatedInput: decision.updatedInput ?? input }
    : { behavior: 'deny', message: decision.reason ?? 'Rejected by human reviewer' };
}
```

---

## 8. Security

1. **Không bao giờ đọc/copy `~/.claude/.credentials.json`**; không set `CLAUDE_CODE_OAUTH_TOKEN`/`ANTHROPIC_API_KEY` trong env agent. CLI tự quản lý refresh. Health-check chỉ qua `claude auth status` (không chứa token).
2. Nếu dùng `options.env` (SDK thay thế toàn bộ env) => build từ `process.env` đã **lọc** (bỏ các secret của backend: DB URL, JWT secret...), giữ `HOME`, `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, `PATH`, `TEMP`, `SystemRoot`.
3. Agent không được truy cập ngoài worktree: `additionalDirectories` tối thiểu, `settings: { permissions: { deny: ["Read(~/.claude/**)", "Read(**/.env*)", "Bash(curl *)"] } }`, cân nhắc `--restricted`/`sandbox` option.
4. Logging redaction: trước khi persist `SDKMessage`/`tool_input` => regex mask `sk-ant-[A-Za-z0-9_-]+`, `Bearer ...`, `"accessToken"`, `"refreshToken"`, nội dung `.env`; không log `transcript_path` content; cắt `tool_result` > N KB.
5. Hook HTTP endpoint (nếu dùng): bind `127.0.0.1`, Bearer token random per boot (như Pixel Agents `server.json`).
6. `-p` **không có trust dialog** và vẫn load `.claude/settings.json` + `.mcp.json` của repo => với repo lạ dùng `settingSources: ['user']` + `strictMcpConfig: true` + `settings: { disableAllHooks: true }`.
7. `maxBudgetUsd`, `maxTurns`, timeout tổng per task; kill process tree khi cancel (SDK `close()` làm việc này).

---

## 9. Đề xuất thiết kế ClaudeCodeAdapter

### 9.1 Interface (NestJS, TypeScript)

> Ghi chú: BrainAdapter/BrainEvent là tên nghiên cứu; tên chính thức là AgentAdapter/AgentEvent (PROVIDER_INTEGRATION §P1, packages/agent-core).

```ts
// src/agents/brains/brain.adapter.ts
export type BrainKind = 'claude-code' | 'codex-cli';

export interface BrainSessionOptions {
  agentId: string;
  cwd: string;                         // git worktree của agent
  additionalDirectories?: string[];
  model: 'haiku' | 'sonnet' | 'opus' | 'fable' | string;
  effort?: 'low' | 'medium' | 'high' | 'xhigh' | 'max';
  systemPromptAppend: string;          // role/persona của AI employee
  allowedTools: string[];              // rule syntax: "Bash(git status *)", "Edit"
  disallowedTools?: string[];
  permissionMode: 'default' | 'acceptEdits' | 'dontAsk' | 'plan' | 'auto';
  maxTurns?: number;
  maxBudgetUsd?: number;
  mcpServers?: Record<string, McpServerConfig>;
  resumeSessionId?: string;            // resume
  forkSession?: boolean;
  sessionId?: string;                  // UUID do ta sinh, để biết trước ID
  outputSchema?: object;               // structured output
}

export type BrainEvent =
  | { type: 'init'; sessionId: string; model: string; tools: string[]; apiKeySource: string }
  | { type: 'text'; text: string; partial?: boolean }
  | { type: 'thinking'; tokens: number }
  | { type: 'tool.start'; toolUseId: string; name: string; input: unknown; parentToolUseId: string | null }
  | { type: 'tool.end'; toolUseId: string; isError: boolean; output: unknown }
  | { type: 'approval.requested'; toolUseId: string; name: string; input: unknown }
  | { type: 'approval.resolved'; toolUseId: string; approved: boolean }
  | { type: 'permission.denied'; name: string; input: unknown }
  | { type: 'rate_limit'; fiveHour: number; sevenDay: number; status: string; resetsAt: number }
  | { type: 'api_retry'; attempt: number; error: string }
  | { type: 'turn.result'; resultIndex: number; text: string; structured?: unknown; isError: boolean;
      numTurns: number; costUsd: number; durationMs: number; usage: Record<string, unknown> }
  | { type: 'exit'; code: number | null; signal: string | null };

export interface ApprovalDecision { approved: boolean; updatedInput?: Record<string, unknown>; reason?: string; }

export interface ApprovalRequest { toolName: string; input: Record<string, unknown>; toolUseId: string; signal: AbortSignal; }

export interface BrainSession {
  readonly sessionId: string;
  readonly events: Observable<BrainEvent>;      // RxJS hoặc EventEmitter
  send(prompt: string | ContentBlock[]): Promise<void>;   // enqueue user turn (streaming input)
  interrupt(): Promise<void>;                              // dừng turn hiện tại, giữ session
  setModel(model: string): Promise<void>;
  setPermissionMode(mode: BrainSessionOptions['permissionMode']): Promise<void>;
  close(): Promise<void>;                                  // graceful: đóng stdin -> grace -> kill
}

export interface BrainAdapter {
  readonly kind: BrainKind;
  healthCheck(): Promise<{ ok: boolean; version: string; loggedIn: boolean; authMethod?: string; subscriptionType?: string }>;
  start(opts: BrainSessionOptions, approver: (req: ApprovalRequest) => Promise<ApprovalDecision>): Promise<BrainSession>;
}
```

### 9.2 Implementation sketch (Agent SDK)

```ts
// src/agents/brains/claude-code.adapter.ts
import { query, type Options, type SDKMessage, type SDKUserMessage } from '@anthropic-ai/claude-agent-sdk';
import { execFile } from 'node:child_process';
import { Subject } from 'rxjs';

@Injectable()
export class ClaudeCodeAdapter implements BrainAdapter {
  readonly kind = 'claude-code' as const;

  async healthCheck() {
    const [ver, status] = await Promise.all([
      execFileP('claude', ['--version']), execFileP('claude', ['auth', 'status']),
    ]);
    const s = JSON.parse(status.stdout);
    return { ok: s.loggedIn === true, version: ver.stdout.trim(), loggedIn: s.loggedIn,
             authMethod: s.authMethod, subscriptionType: s.subscriptionType };
  }

  async start(opts: BrainSessionOptions, approver): Promise<BrainSession> {
    const events = new Subject<BrainEvent>();
    const abort = new AbortController();
    const input = new AsyncQueue<SDKUserMessage>();   // push-based async iterable; close() => end stream

    const options: Options = {
      cwd: opts.cwd, additionalDirectories: opts.additionalDirectories,
      model: opts.model, effort: opts.effort,
      systemPrompt: { type: 'preset', preset: 'claude_code', append: opts.systemPromptAppend },
      allowedTools: opts.allowedTools, disallowedTools: opts.disallowedTools,
      permissionMode: opts.permissionMode, permissionPrompts: 'host',
      maxTurns: opts.maxTurns, maxBudgetUsd: opts.maxBudgetUsd,
      mcpServers: opts.mcpServers, strictMcpConfig: true,
      settingSources: ['user'], settings: { disableAllHooks: true },
      resume: opts.resumeSessionId, forkSession: opts.forkSession, sessionId: opts.sessionId,
      outputFormat: opts.outputSchema ? { type: 'json_schema', schema: opts.outputSchema } : undefined,
      includePartialMessages: true, abortController: abort,
      pathToClaudeCodeExecutable: process.env.AIW_CLAUDE_BIN,   // optional: dùng CLI đã cài thay vì bundled
      env: sanitizedEnv(),                                       // KHÔNG chứa secret backend; KHÔNG chứa API key
      stderr: (d) => this.logger.debug(redact(d)),
      canUseTool: async (toolName, toolInput, { signal, toolUseID }) => {
        events.next({ type: 'approval.requested', toolUseId: toolUseID, name: toolName, input: redact(toolInput) });
        const d = await approver({ toolName, input: toolInput, toolUseId: toolUseID, signal });
        events.next({ type: 'approval.resolved', toolUseId: toolUseID, approved: d.approved });
        return d.approved ? { behavior: 'allow', updatedInput: d.updatedInput ?? toolInput }
                          : { behavior: 'deny', message: d.reason ?? 'Rejected by reviewer' };
      },
      hooks: {
        PreToolUse: [{ hooks: [async (inp) => policyGuard(inp, opts) /* {} hoặc deny JSON */] }],
      },
    };

    const q = query({ prompt: input, options });
    let sessionId = opts.sessionId ?? '';

    (async () => {
      try {
        for await (const m of q) { sessionId = m.session_id ?? sessionId; for (const e of mapMessage(m)) events.next(e); }
        events.next({ type: 'exit', code: 0, signal: null });
      } catch (e) { events.error(e); }
      finally { events.complete(); }
    })();

    return {
      get sessionId() { return sessionId; }, events: events.asObservable(),
      send: async (prompt) => input.push({ type: 'user',
        message: { role: 'user', content: typeof prompt === 'string' ? [{ type: 'text', text: prompt }] : prompt },
        parent_tool_use_id: null, session_id: sessionId }),
      interrupt: () => q.interrupt().then(() => undefined),
      setModel: (m) => q.setModel(m), setPermissionMode: (p) => q.setPermissionMode(p),
      close: async () => { input.close(); await withTimeout(drained(events), 3000).catch(() => abort.abort()); q.close(); },
    };
  }
}

function mapMessage(m: SDKMessage): BrainEvent[] {
  switch (m.type) {
    case 'system':
      return m.subtype === 'init'
        ? [{ type: 'init', sessionId: m.session_id, model: m.model, tools: m.tools, apiKeySource: m.apiKeySource }] : [];
    case 'stream_event':
      return m.event.type === 'content_block_delta' && m.event.delta.type === 'text_delta'
        ? [{ type: 'text', text: m.event.delta.text, partial: true }] : [];
    case 'assistant':
      return m.message.content.flatMap(b =>
        b.type === 'text' ? [{ type: 'text', text: b.text }]
        : b.type === 'tool_use' ? [{ type: 'tool.start', toolUseId: b.id, name: b.name, input: redact(b.input), parentToolUseId: m.parent_tool_use_id }]
        : []);
    case 'user':
      return m.message.content.filter(b => b.type === 'tool_result')
        .map(b => ({ type: 'tool.end', toolUseId: b.tool_use_id, isError: !!b.is_error, output: truncate(redact(b.content)) }));
    case 'rate_limit_event': {
      const w = m.rate_limit_info.unifiedWindows;
      return [{ type: 'rate_limit', fiveHour: w.five_hour.utilization, sevenDay: w.seven_day.utilization,
                status: m.rate_limit_info.status, resetsAt: m.rate_limit_info.resetsAt }];
    }
    case 'result':
      return [{ type: 'turn.result', resultIndex: m.result_index ?? 0, text: m.subtype === 'success' ? m.result : '',
                structured: (m as any).structured_output, isError: m.is_error, numTurns: m.num_turns,
                costUsd: m.total_cost_usd, durationMs: m.duration_ms, usage: m.usage }];
    default: return [];
  }
}
```

### 9.3 Process lifecycle

```text
 ┌──────────┐  start(opts)   ┌────────────────┐ spawn claude (bundled)  ┌──────────────────┐
 │ Scheduler│ ─────────────► │ ClaudeCode     │ ───────────────────────►│ claude subprocess│
 │ (max 10) │                │ Adapter/Session│ ◄── stream-json ─────── │ cwd = worktree   │
 └──────────┘                └────────────────┘                          └──────────────────┘
      │ send(prompt)   → input queue → stdin {"type":"user",...}
      │ events         ← init / text / tool.start / tool.end / approval.requested / rate_limit / turn.result
      │ approval       ← canUseTool Promise pending → UI Approve/Deny → resolve → tool chạy / tool_result is_error
      │ interrupt()    → control_request interrupt (giữ session, turn hiện tại dừng)
      │ close()        → đóng stdin (EOF) → CLI xả output, chạy SessionEnd hooks → exit 0
      │                  nếu > grace (3s) → abortController.abort() / q.close() → SIGTERM → exit 143 (turn dang dở không ghi result)
      │ crash/exit≠0   → events 'exit' → Scheduler đánh dấu agent lỗi → resume bằng sessionId sau
      │                  (CLAUDE_CODE_RESUME_INTERRUPTED_TURN=1 nếu muốn tiếp turn dang dở)
      │ resume         → start({ resumeSessionId }) hoặc forkSession: true để tạo nhánh thử nghiệm
      │ cleanup        → git worktree remove; transcript vẫn ở ~/.claude/projects/<slug>/ (audit, cleanupPeriodDays=30)
```

Quy tắc vận hành:

1. **1 agent = 1 process sống lâu** (streaming input) trong lúc làm task; idle > N phút => `close()` để giải phóng RAM và resume bằng `sessionId` khi có task mới.
2. Semaphore `MAX_PARALLEL_AGENTS=10` [số nghiên cứu lịch sử; mặc định khuyến nghị: D-010 APPROVED, global 5, trần 18]; đọc `rate_limit` events => throttle toàn cục; `seven_day > 0.9` => chỉ cho lead chạy.
3. Mỗi `turn.result` ghi `costUsd`, `usage`, `numTurns` vào DB per agent/task để báo cáo "mức tiêu thụ" (ước tính list price).
4. Cancel do người dùng: `interrupt()` trước, 5s sau mới `close()`; không bao giờ kill -9 trực tiếp (để SessionEnd hook và process tree của Bash được dọn). Docs: SIGTERM => exit 143, "leaves the turn that was in progress unfinished and records no result"; SIGINT hoặc `interrupt()` mới kết thúc turn sạch.
5. Fallback raw CLI (`spawn('claude', ['-p', '--input-format', 'stream-json', '--output-format', 'stream-json', '--verbose', '--session-id', id, ...])`, prompt qua stdin) chỉ dùng cho healthcheck hoặc khi SDK binary không khớp; parse NDJSON bằng `readline`, dùng chung `mapMessage`.
6. Codex CLI adapter (Agent khác nghiên cứu) implement cùng `BrainAdapter` interface để scheduler không phân biệt brain.

### 9.4 Checklist triển khai

- [ ] `npm i @anthropic-ai/claude-agent-sdk@^0.3.314 zod@^4` trong NestJS app; pin version, test lại khi nâng.
- [ ] `ClaudeCodeAdapter.healthCheck()` chạy lúc boot: fail-fast nếu `loggedIn=false` => hướng dẫn user chạy `claude auth login` (tương tác, ngoài app).
- [ ] `ApprovalService` + WebSocket gateway cho `approval.requested`.
- [ ] `RateLimitGuard` đọc `rate_limit` event, lưu utilization mới nhất, chặn scheduler.
- [ ] `WorktreeService` tạo/xóa `git worktree` per agent-task.
- [ ] Redaction middleware trước mọi logger/DB write.
- [ ] E2E test: PONG, resume, canUseTool deny, 10 session song song với `haiku --max-turns 1`.
