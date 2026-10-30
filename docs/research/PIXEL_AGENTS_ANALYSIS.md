# Phân tích Pixel Agents (v1.5.0, MIT) — Tham chiếu cho AI WORKSPACE

- Repo: `D:\Code\Tuyen\Agent management\pixel-agents` (clone https://github.com/pixel-agents-hq/pixel-agents)
- Tác giả: Pablo De Lucca, license MIT (code). Ngày phân tích: 2026-10-30
- Người phân tích: Agent 02 — Pixel Agents Research Analyst
- Mục tiêu: xác định cơ chế quan sát Claude Code, abstraction Provider/Adapter, engine render 2D office, asset pipeline, persistence, và đánh giá phần nào tái sử dụng được cho AI WORKSPACE (NestJS + React, Phaser hoặc Canvas).

---

## 0. Tóm tắt điều hành (TL;DR)

1. Pixel Agents là **VS Code extension + standalone CLI (`npx pixel-agents`)** dùng chung một source tree: `core/` (contract, interface) → `server/` (Fastify runtime, provider Claude) → `adapters/vscode/` và `webview-ui/` (React 19 + **Canvas 2D thuần, không Phaser**).
2. Quan sát Claude Code bằng **2 đường song song**: (a) **Hooks mode** — cài 12 hook event vào `~/.claude/settings.json`, mỗi event chạy `node ~/.pixel-agents/hooks/claude-hook.js` đọc JSON từ stdin rồi **HTTP POST** tới `http://127.0.0.1:<port>/api/hooks/claude` (Bearer token, discovery qua `~/.pixel-agents/servers/*.json`); (b) **Heuristic mode** — tail-poll file JSONL transcript tại `~/.claude/projects/<path-hash>/<session-id>.jsonl` mỗi 500 ms, parse `assistant/tool_use`, `user/tool_result`, `system/turn_duration`, `progress`. JSONL **luôn** được đọc kể cả khi có hooks (để lấy nội dung tool, context usage, sub-agent).
3. Abstraction `HookProvider` (`core/src/provider.ts`) chuẩn hoá mọi CLI thành `AgentEvent` 10 kind (`toolStart`, `toolEnd`, `turnEnd`, `subagentStart`, `subagentEnd`, `subagentTurnEnd`, `progress`, `permissionRequest`, `sessionStart`, `sessionEnd`). Runtime chỉ dispatch theo `kind`. Thêm CLI mới = 1 thư mục `server/src/providers/hook/<id>/`. Hiện chỉ có Claude; Codex/Gemini/Cursor là roadmap.
4. Engine office: tile 16 px, grid tối đa 64×64, BFS 4 hướng, FSM nhân vật 3 state (`idle/walk/type`), z-sort theo Y, sprite cache offscreen canvas theo zoom, DOM overlay cho label + bubble. Layout editor đầy đủ (floor/wall/carpet/furniture/area/pet, undo 50 bước, import/export JSON).
5. Asset: 6 character sheet 112×96 (7 frame × 3 hướng, 16×32/frame), 25 furniture với `manifest.json` (rotation/state/animation group), floor 16×16, wall 64×128 autotile 4-bit, carpet 64×64 marching squares, pet 96×96. **Nhân vật dựa trên JIK-A-4 "Metro City" free pack (itch.io)** — license art KHÔNG được ghi riêng trong repo, cần xác minh trước khi dùng thương mại.
6. Khuyến nghị: tái sử dụng trực tiếp `core/src/provider.ts` + `claude.ts` (normalize) + `claude-hook.ts` + `transcriptParser.ts` + `fileWatcher.ts` (phần tail), engine pathfinding/FSM/renderer/sprite cache/colorize, và format manifest asset. Viết mới: multi-tenant state, Codex provider, team/task domain, Phaser wrapper (nếu chọn Phaser).

---

## 1. Kiến trúc tổng thể

### 1.1 Layering (từ `CLAUDE.md` mục Architecture)

```
core/        Protocol + interface, zero side effects (asyncapi.yaml, provider.ts, transport.ts, adapter.ts, schemas.ts)
server/      AgentRuntime, AgentStateStore, Fastify HTTP/WS, providers/hook/claude, persistence, transcript scan, CLI
adapters/vscode/  WebviewViewProvider, terminal lifecycle (vscode.Terminal), state migration
webview-ui/  React 19 + Vite + Canvas 2D; transport PostMessage (VS Code) hoặc WebSocket (standalone)
```

Quy tắc phụ thuộc nghiêm ngặt: `core` không phụ thuộc gì; `server` → `core`; `webview-ui` → `core`; `adapters/vscode` → `core + server`. CLI standalone không bao giờ import `adapters/vscode`.

### 1.2 Luồng dữ liệu (hub-and-spoke)

```
Hook script ──POST /api/hooks/:providerId──┐
                                           ├─→ HookProvider.normalizeHookEvent() ─→ AgentEvent
JSONL transcript ──FileWatcher──→ TranscriptParser ┘
                                           ↓
                               AgentRuntime (dispatch theo event.kind)
                                           ↓
                               AgentStateStore (EventEmitter, single source of truth)
                                           ↓
                               StoreEvents → broadcast ServerMessage
                                           ↓
                 PostMessageTransport (VS Code)  |  WebSocketTransport (/ws, standalone)
```

File quan trọng:
- `server/src/agentRuntime.ts` (579 dòng): timer maps, scanner (project 1 s, external 3 s, stale 30 s), `HookEventHandler`, `SessionRouter`, `DismissalTracker`.
- `server/src/agentStateStore.ts`: typed mutations + events `agentAdded/agentRemoved/agentUpdated/broadcast`. Không module nào trong `server/` gọi transport trực tiếp.
- `server/src/httpServer.ts`: Fastify v5, route `POST /api/hooks/:providerId` (Bearer), `GET /api/health`, `GET /ws`, `GET /*` (SPA, standalone).
- `server/src/clientMessageHandler.ts` (721 dòng): dispatch 27 `ClientMessage` từ webview; gate `refuseReadOnly` cho client không có token.

### 1.3 Wire protocol — AsyncAPI 3.0

`core/asyncapi.yaml` là contract duy nhất; `core/src/messages.ts` sinh tự động bằng `@asyncapi/modelina` (CI kiểm tra drift). 39 ServerMessage + 27 ClientMessage, discriminator `type`. Các message cốt lõi liên quan activity:

```
agentCreated, agentClosed, agentStatus{status:'active'|'waiting', awaitingInput?},
agentToolStart{id, toolId, status, toolName?, runInBackground?, permissionActive?},
agentToolDone, agentToolsClear, agentToolPermission, agentToolPermissionClear,
subagentToolStart/Done/Permission, subagentClear, agentTeamInfo, agentContextUsage,
existingAgents, layoutLoaded, characterSpritesLoaded, floorTilesLoaded, wallTilesLoaded,
carpetTilesLoaded, furnitureAssetsLoaded, petSpritesLoaded, providerCapabilities,
hooksStatus, hooksConsentRequest, officeAccess, directoriesLoaded, terminalSessionOpened...
```

`core/src/messages.ts:126`: `export type AgentActivityStatus = 'active' | 'waiting';` — trạng thái trên dây chỉ có 2 giá trị; "Done" / "Waiting for input" / "Permission request" được suy ra từ `awaitingInput` và bubble.

### 1.4 Transport abstraction (`core/src/transport.ts`)

```ts
export interface MessageTransport {
  send(msg: ClientMessage): void;
  onMessage(handler: (msg: ServerMessage) => void): () => void;
  readonly ready: Promise<void>;
  readonly state: TransportState; // 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  onStateChange(handler: (state: TransportState) => void): () => void;
  dispose(): void;
}
```

`webview-ui/src/transport/index.ts` `createTransport()` là điểm rẽ nhánh duy nhất. `WebSocketTransport` backoff 250 ms → 4 s, queue send khi disconnect. Rất phù hợp để AI WORKSPACE dùng lại gần như nguyên vẹn (thay `ClientMessage/ServerMessage` bằng schema của ta).

### 1.5 Build

- `esbuild.js`: bundle `adapters/vscode/extension.ts` → `dist/extension.js`; `server/src/cli.ts` → `dist/cli.js` (external fastify, node-pty); hook script `claude-hook.ts` → `dist/hooks/claude-hook.js` (CJS + shebang, target node18); copy `webview-ui/public/assets` → `dist/assets`.
- `webview-ui/vite.config.ts`: plugin `browser-mock-assets` decode PNG (pngjs) ở build-time thành JSON sidecar để standalone browser fetch được sprite không cần server VS Code.
- Dependencies chính: `fastify`, `@fastify/websocket`, `@fastify/static`, `@fastify/cors`, optional `@lydell/node-pty`; webview: `react 19`, `@xterm/xterm`, `tailwindcss 4`, `pngjs` (dev).

---

## 2. Cơ chế phát hiện và theo dõi Claude Code session

### 2.1 Hooks mode (mặc định, instant)

**Cài đặt** (`server/src/providers/hook/claude/claudeHookInstaller.ts`, 749 dòng): ghi vào `~/.claude/settings.json` một entry cho mỗi event trong danh sách:

```ts
// server/src/providers/hook/claude/constants.ts
export const CLAUDE_HOOK_EVENTS = [
  'SessionStart','SessionEnd','Stop','PermissionRequest','Notification',
  'PreToolUse','PostToolUse','PostToolUseFailure',
  'SubagentStart','SubagentStop','TeammateIdle','TaskCompleted',
] as const;
```

Entry có dạng `{ matcher: '', hooks: [{ type: 'command', command: 'node "~/.pixel-agents/hooks/claude-hook.js"', timeout: 5 }] }` (`makeHookEntry`, dòng 536). `UserPromptSubmit` và `TaskCreated` **cố ý không cài** để không thu thập prompt text. Installer có: consent gate trước lần ghi đầu, backup `.pixel-agents.backup` một lần, atomic tmp+rename, retry 3 lần nếu file bị Claude Code ghi đồng thời, nhận diện hook của mình bằng suffix `/.pixel-agents/hooks/claude-hook.js` (case-insensitive), không bao giờ rewrite shape mà mình không tạo.

**Hook script** (`server/src/providers/hook/claude/hooks/claude-hook.ts`, 198 dòng): đọc toàn bộ stdin → JSON; đọc registry `~/.pixel-agents/servers/*.json` (`{port, pid, token, debugLog?}`), bỏ entry có PID chết; fan-out POST tới mọi server còn sống (fallback `~/.pixel-agents/server.json`); mọi lỗi nuốt im lặng, luôn `exit 0`, timeout 2 s:

```ts
const req = http.request({
  hostname: '127.0.0.1', port: server.port,
  path: `${HOOK_API_PREFIX}/claude`, method: 'POST',
  headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${server.token}` },
  timeout: 2000,
}, ...);
```

**Chuẩn hoá** (`claude.ts` `normalizeHookEvent`): bảng ánh xạ Claude event → `AgentEvent`:

| Claude hook event | AgentEvent | Ghi chú |
|---|---|---|
| `PreToolUse` | `toolStart{toolId:'hook-<ts>', toolName, input, runInBackground}` | id tạm vì payload không có tool_use id; id thật tới sau qua JSONL |
| `PostToolUse` / `PostToolUseFailure` | `toolEnd{toolId:'current'}` | handler tự correlate với `currentHookToolId` |
| `Stop` | `turnEnd` | "Done" |
| `Notification(idle_prompt)` | `turnEnd{awaitingInput:true}` | "Waiting for input" |
| `PermissionRequest`, `Notification(permission_prompt)` | `permissionRequest` | bubble "..." |
| `SessionStart` | `sessionStart{source, transcriptPath, cwd}` | dùng để adopt session ngoài |
| `SessionEnd` | `sessionEnd{reason}` | |
| `SubagentStart/Stop` | `subagentStart/End` | |
| `TeammateIdle` / `TaskCompleted` | `subagentTurnEnd{reason:'idle'|'completed'}` | Agent Teams |
| `UserPromptSubmit`, `TaskCreated` | `null` | bị bỏ qua |

### 2.2 Heuristic mode (fallback) — JSONL transcript

- Đường dẫn: `~/.claude/projects/<project-hash>/<session-id>.jsonl`; `project-hash` = đường dẫn workspace với `:`, `\`, `/` → `-` (`core/src/normalizeProjectPath.ts`). `claude.ts getSessionDirs()` có fallback case-insensitive cho Windows (drive letter).
- `server/src/fileWatcher.ts` (1573 dòng): mỗi agent 1 `setInterval` 500 ms (`FILE_WATCHER_POLL_INTERVAL_MS`), `readNewLines()` đọc từ `agent.fileOffset` bằng `fs.readSync`, giữ `lineBuffer` cho dòng chưa kết thúc. Tác giả ghi rõ "fs.watch unreliable on Windows/macOS/WSL2 — always pair with polling". Scanner thư mục dự án 1 s để adopt file JSONL mới; scanner toàn cục 3 s (`Watch All Sessions`, chỉ file > 3 KB và mtime < 10 phút); stale check 30 s.
- `/clear` tạo file JSONL mới: phát hiện bằng nội dung `"/clear</command-name>"` trong 8 KB đầu file.
- `server/src/transcriptParser.ts` (741 dòng) `processTranscriptLine()` xử lý từng record:

```ts
if (record.type === 'assistant' && Array.isArray(assistantContent)) {
  const hasToolUse = blocks.some((b) => b.type === 'tool_use');
  if (hasToolUse) {
    agents.broadcast({ type: 'agentStatus', id: agentId, status: 'active' });
    for (const block of blocks) if (block.type === 'tool_use' && block.id) {
      const status = formatToolStatus(toolName, block.input || {});
      agent.activeToolIds.add(block.id); agent.activeToolStatuses.set(block.id, status); ...
```

  - `user` + `tool_result` → tool done (delay 300 ms `TOOL_DONE_DELAY_MS` để React không nuốt state ngắn).
  - `system` + `subtype:"turn_duration"` → turn end chắc chắn (~98% turn có tool); text-only turn dùng timer 5 s (`TEXT_IDLE_DELAY_MS`).
  - `progress` + `data.type`: `agent_progress` (sub-agent tool), `bash_progress`/`mcp_progress` (tool đang chạy → reset permission timer 7 s).
  - `message.usage` trên record assistant → `agentContextUsage` (snapshot `input + cache_creation + cache_read + output`, cửa sổ do provider quyết: 1M / 200k).
  - Team metadata (`record.teamName`, `record.agentName`) → `agentTeamInfo`.
- Timer heuristic: permission 7 s (`PERMISSION_TIMER_DELAY_MS`) khi tool không exempt chạy mà không có data; bị tắt khi `hookDelivered=true`.

### 2.3 Ánh xạ sang trạng thái nhân vật

Server chỉ gửi `agentStatus active|waiting` + `agentToolStart/Done` + `agentToolPermission`. Webview (`useExtensionMessages.ts`) chuyển thành:

```ts
} else if (msg.type === 'agentToolStart') {
  const toolName = (msg.toolName as string | undefined) ?? extractToolName(status);
  os.setAgentTool(id, toolName);
  os.setAgentActive(id, true);
  if (!permissionActive) os.clearPermissionBubble(id);
...
} else if (msg.type === 'agentStatus') {
  os.setAgentActive(id, status === 'active');
  if (status === 'waiting') { os.showWaitingBubble(id, msg.awaitingInput === true); playDoneSound(); }
```

Bảng trạng thái nhân vật kết quả:

| Trạng thái domain (CONTEXT.md) | Tín hiệu | Nhân vật |
|---|---|---|
| Active, tool "write-like" (Write/Edit/Bash/Task) | `toolStart` | đi về ghế, `TYPE` state, frame typing 2 khung (0.3 s) |
| Active, tool "read-like" (Read/Grep/Glob/WebFetch/WebSearch) | `toolStart` + `readingTools` | `TYPE` state nhưng dùng frame reading |
| Thinking (không có tool) | không có tín hiệu riêng | vẫn `active` cho tới turn end |
| Done | `turnEnd` (Stop / turn_duration) | bubble check xanh fade 2 s, sau đó wander |
| Waiting for input | `turnEnd{awaitingInput}` | bubble check + label "Waiting for input", ngồi yên |
| Permission request | `permissionRequest` / timer 7 s | bubble "..." vàng, giữ tới khi clear |
| Sub-agent | `agentToolStart` tool ∈ `subagentToolNames` | nhân vật phụ id âm, cùng palette, đứng cạnh parent |

`webview-ui/src/office/toolUtils.ts`: `providerCapabilities` message mang `readingTools` + `subagentToolNames` từ provider xuống webview — webview không hardcode tên tool Claude.

### 2.4 Launch & adoption

- VS Code (`adapters/vscode/agentManager.ts:64-81`): `vscode.window.createTerminal({ name: 'Claude Code #n' })` rồi `terminal.sendText('claude --session-id <uuid> [--dangerously-skip-permissions]')`. Session ID tạo trước bằng `crypto.randomUUID()` để biết chính xác file JSONL.
- Standalone (`server/src/terminal/standaloneAgentLauncher.ts` + `ptySessionManager.ts`): spawn qua `node-pty`, mirror scrollback 5000 dòng bằng xterm headless, stream qua WS `/terminal/:agentId` tới xterm.js trong browser.
- Adopt session ngoài: qua `SessionStart` hook (có `transcript_path`, `cwd`) hoặc scanner JSONL; `SessionRouter` buffer event 5 s cho tới khi agent đăng ký.

---

## 3. Provider abstraction — thiết kế cho đa CLI

### 3.1 `HookProvider` (`core/src/provider.ts`, 151 dòng)

```ts
export interface HookProvider {
  readonly kind: 'hook'; readonly id: string; readonly displayName: string;
  readonly protocolVersion: number;
  normalizeHookEvent(raw: Record<string, unknown>): { sessionId: string; event: AgentEvent } | null;
  installHooks(serverUrl: string, authToken: string): Promise<void>;
  uninstallHooks(): Promise<void>;
  areHooksInstalled(): Promise<boolean>;
  consentDisclosure(): { headline: string; disclosure: string };
  formatToolStatus(toolName: string, input?: unknown): string;
  readonly permissionExemptTools: ReadonlySet<string>;
  readonly subagentToolNames: ReadonlySet<string>;
  readonly readingTools: ReadonlySet<string>;
  readonly terminalNamePrefix?: string;
  contextWindowForModel?(model: string | undefined): number | undefined;
  // file fallback
  getSessionDirs?(workspacePath: string): string[];
  getAllSessionRoots?(): string[];
  readonly sessionFilePattern?: string;
  parseTranscriptLine?(line: string): AgentEvent | null;
  buildLaunchCommand?(sessionId, cwd, opts?): { command; args; env? };
  readonly team?: TeamProvider;
}
```

Claude implementation (`claude.ts`):

```ts
export const claudeProvider: HookProvider = {
  kind: 'hook', id: 'claude', displayName: 'Claude Code', protocolVersion: 1,
  permissionExemptTools: new Set(['Task', 'Agent', 'AskUserQuestion']),
  subagentToolNames: new Set(['Task', 'Agent']),
  readingTools: new Set(['Read', 'Grep', 'Glob', 'WebFetch', 'WebSearch']),
  sessionFilePattern: '*.jsonl', team: claudeTeamProvider, ...
};
```

### 3.2 `TeamProvider` (`core/src/teamProvider.ts`)

Semantic queries: `isTeammateSpawnCall`, `extractTeammateNameFromEvent`, `discoverTeammates(projectDir, leadSessionId, teamName?)`, `extractTeammateSpawnFromToolResult`, `getTeamMetadataForSession`, `extractTeamMetadataFromRecord`, `getTeamMembers`. Claude đọc `~/.claude/teams/<name>/config.json`. Phân loại Sub-agent vs Teammate chỉ dựa vào **có tên hay không** (sidecar `name` trong `<projectDir>/<leadSessionId>/subagents/`). Sub-agent không tên được theo dõi trong **shadow `AgentStateStore`** (`server/src/subagentWatch.ts`) rồi dịch sang `subagentToolStart/Done` trên store chính.

### 3.3 Đánh giá cho Provider Adapter của AI WORKSPACE

- Điểm mạnh: boundary rõ, runtime hoàn toàn không biết tên tool Claude; provider tự khai báo taxonomy (`readingTools`, `subagentToolNames`) và cửa sổ context. Mô hình "hook push + transcript pull" bổ sung nhau — hook cho độ trễ thấp, transcript cho nội dung chi tiết.
- Hạn chế: (1) chỉ có `kind: 'hook'`; `FileProvider`/`StreamProvider` chưa tồn tại (TODO trong file); (2) nhiều logic Claude-specific vẫn rò rỉ vào `transcriptParser.ts`/`fileWatcher.ts` (record `type`, `tool_use`, `turn_duration`, `agent_progress`, sidecar) — `parseTranscriptLine` optional của interface **không** được dùng, parser gọi thẳng cấu trúc Claude; (3) correlation `toolId:'current'`/`hook-<ts>` là workaround vì hook payload Claude thiếu tool_use id; (4) state là per-process, một `AgentStateStore` toàn cục, không multi-tenant.
- Với Codex CLI: Codex không có hook API tương đương; cần provider dạng stream (`codex exec --json` / app-server protocol JSON-RPC) → chính là `StreamProvider` mà Pixel Agents chưa viết. Ta nên thiết kế `AgentEvent` union tương tự nhưng thêm `thinking`, `message`, `tokenUsage`, `error`, và bỏ `'current'` sentinel bằng cách bắt buộc provider tự correlate id.

---

## 4. Engine render 2D office

### 4.1 Công nghệ: Canvas 2D thuần (không Phaser)

- `webview-ui/src/office/engine/gameLoop.ts`: `requestAnimationFrame`, `dt` cap 0.1 s (`MAX_DELTA_TIME_SEC`), `imageSmoothingEnabled=false` mỗi frame.
- `OfficeCanvas.tsx` (823 dòng): `canvas.width = round(rect.width * devicePixelRatio)`, không `ctx.scale(dpr)`; zoom = số nguyên device-pixel / sprite-pixel (1–10, mặc định `round(2*dpr)`); `ResizeObserver`; pan bằng chuột giữa, touch pan/pinch (`useCanvasTouchGestures.ts`); hit-test `officeState.getCharacterAt(worldX, worldY)`; camera follow lerp 0.1.
- Game state nằm trong class imperative `OfficeState` (`officeState.ts`, 1229 dòng), **không** phải React state; React chỉ render toolbar/overlay. Đây là mẫu đúng để tránh re-render.

### 4.2 Tile map & layout model (`webview-ui/src/office/types.ts`)

```ts
export const TileType = { WALL: 0, FLOOR_1: 1, ... FLOOR_9: 9, VOID: 255 } as const;
export interface OfficeLayout {
  version: 1; cols: number; rows: number;
  tiles: TileType[];                       // row-major, cols*rows
  furniture: PlacedFurniture[];            // { uid, type, col, row, color? }
  tileColors?: Array<ColorValue | null>;   // { h, s, b, c, colorize? } per tile
  layoutRevision?: number;
  pets?: PlacedPet[];                      // { id, petType }
  carpetTiles?: Array<CarpetTile | null>;  // { variant, color?, accentColor?, order? }
  areas?: AreaDefinition[];                // { label, color }
  areaTiles?: Array<string | null>;        // area label per tile
}
```

`TILE_SIZE = 16`, mặc định 20×11, tối đa 64×64 (`MAX_COLS/ROWS`). Default layout `webview-ui/public/assets/default-layout-1.json` 21×22, 36 furniture. Lưu `~/.pixel-agents/layout.json` (atomic tmp+rename, watcher fs.watch + poll 2 s, `markOwnWrite()`).

### 4.3 Pathfinding & movement (`layout/tileMap.ts`, 105 dòng)

BFS 4-hướng trên grid, `isWalkable` = không WALL/VOID và không trong `blockedTiles` (set `"col,row"` từ footprint furniture, trừ background rows và ghế của chính mình `withOwnSeatUnblocked`). Trả path không gồm start. Di chuyển tuyến tính tile→tile `WALK_SPEED_PX_PER_SEC = 48`, walk frame 0.15 s (4 khung), type frame 0.3 s (2 khung). Không có A*, không diagonal, không tránh va chạm giữa nhân vật — đủ cho map 64×64.

### 4.4 Character state machine (`engine/characters.ts`, 342 dòng)

```
TYPE  --(!isActive, seatTimer hết)--> IDLE
IDLE  --(isActive && có ghế)--> WALK(path→seat) | --(isActive, không ghế)--> TYPE tại chỗ
IDLE  --(wanderTimer hết)--> WALK(random walkable) ; sau wanderLimit(3–6) lần → về ghế nghỉ 120–240 s
WALK  --(đến đích)--> TYPE (nếu đích là ghế) | IDLE
WALK  --(isActive giữa chừng)--> repath về ghế
```

Sprite chọn trong `getCharacterSprite()`: `TYPE` → `reading[dir][frame%2]` nếu `isReadingTool(currentTool)` ngược lại `typing[dir][frame%2]`; `WALK` → `walk[dir][frame%4]`; `IDLE` → `walk[dir][1]`. Trái = flip phải. Ngồi dịch xuống 6 px (`CHARACTER_SITTING_OFFSET_PX`). `setAgentActive(false)` đặt `seatTimer=-1` sentinel để bỏ qua rest dài khi turn vừa kết thúc. Hiệu ứng spawn/despawn "matrix rain" 0.3 s (`matrixEffect.ts`).

Pet (`petEntity.ts`): FSM `IDLE/WALK/FOLLOW`, theo nhân vật trong bán kính N tile với xác suất `PET_FOLLOW_CHANCE`, bubble trái tim khi click.

### 4.5 Renderer (`engine/renderer.ts`, 1014 dòng) — thứ tự vẽ trong `renderFrame()`

1. `clearRect` → `mapOffset()` (center + pan, số nguyên)
2. `renderTileGrid` (floor colorized theo `tileColors`)
3. `renderCarpetLayer` (marching squares)
4. `renderAreaOverlay` (edit mode)
5. `renderSeatIndicators` (khi chọn nhân vật)
6. `renderScene`: gom furniture + wall instances + characters + pets vào `drawables[]`, **sort theo `zY`** rồi vẽ. Character `zY = y + 8 + 0.5`; ghế quay lưng `zY = (row+1)*16 + 1` để lưng ghế che nhân vật; surface item `zY = max(spriteBottom, deskZY + 0.5)`.
7. `renderPetBubbles`, `renderAreaLabels`
8. Edit overlays: grid, ghost border mở rộng, ghost preview (xanh/đỏ), selection, nút delete/rotate.

Speech bubble của nhân vật và activity label **vẽ bằng DOM** (`ToolOverlay.tsx`, 378 dòng) chồng lên canvas, dùng `projection.ts overlayProjection()` để đổi world→screen; lý do: bubble phải nằm trên label panel và label cần font/wrap của HTML. Sprite bubble 11×13 lưu JSON (`sprites/bubble-*.json`, palette + pixels).

### 4.6 Sprite pipeline

- `SpriteData = string[][]` (mảng 2D hex `#RRGGBB` / `#RRGGBBAA` / `''` = trong suốt). PNG → SpriteData ở server/build-time bằng pngjs (`core/src/assets/pngDecoder.ts`, alpha ≥ 2 là hiển thị).
- `sprites/spriteCache.ts`: `getCachedSprite(sprite, zoom)` vẽ từng pixel bằng `fillRect` vào offscreen canvas, cache `Map<zoom, WeakMap<SpriteData, HTMLCanvasElement>>`; `getOutlineSprite` sinh viền trắng 1 px cho selection.
- `colorize.ts`: 2 chế độ — Colorize (grayscale → luminance → contrast → brightness → HSL cố định, dùng cho floor/carpet) và Adjust (dịch HSL gốc, dùng cho furniture và hue shift nhân vật). Palette 6 skin; agent thứ 7+ dùng skin lặp + hue shift 45–315°, gán ở server (`paletteAssigner.ts`) để mọi client thấy giống nhau.
- Wall (`wallTiles.ts`): sheet 64×128 = 4×4 ô 16×32, bitmask 4-bit N=1 E=2 S=4 W=8, sprite cao 16 px hơn tile (mặt 3D), z-sort cùng furniture.
- Carpet (`carpetTiles.ts`): sheet 64×64 = 16 case marching squares (NW=1 NE=2 SE=4 SW=8), dual-color (main/accent theo luminance).
- Furniture auto-state: khi agent active ngồi hướng về bàn, điện tử trong 3 tile sâu + 1 tile hai bên chuyển sang sprite `_ON_*` và chạy animation frame 0.2 s (`rebuildFurnitureInstances`), không sửa layout.

### 4.7 Seat & desk assignment

`layoutSerializer.ts layoutToSeats()`: mỗi tile footprint (trừ background rows) của furniture `category:'chairs'` = 1 seat (`uid`, `uid:1`, ...). Hướng ngồi: (1) `orientation` của ghế (front→DOWN, back→UP, side→LEFT/RIGHT) (2) hướng có desk kề (3) DOWN. `OfficeState.addAgent()` chọn ghế: `preferredSeatId` (persist) → `closestFreeSeat(anchor)` (teammate ngồi cạnh lead, Manhattan) → `findFreeSeat(directoryName)` ưu tiên seat trong Area được map với Directory. Reassign bằng click nhân vật → click ghế trống; persist qua `saveAgentSeats` → `~/.pixel-agents/<ns>-state.json`.

### 4.8 Layout editor (`office/editor/`)

- Tool: `SELECT, TILE_PAINT, WALL_PAINT, ERASE, FURNITURE_PLACE, FURNITURE_PICK, COLOR_PICK, EYEDROPPER, CARPET_PAINT, CARPET_PICK, AREA_PAINT, PETS`.
- `editorActions.ts` (796 dòng) thuần hàm: `canPlaceFurniture`, `dropFurniture`, `expandLayout`, `removeFurnitureAt`, `getSurfaceRiders` (bàn kéo theo đồ trên mặt bàn)…; `editorState.ts` imperative: ghost, selection, drag, undo/redo 50 bước theo stroke session; `editorRenderState.ts` build ghosts cho renderer.
- Phím: R xoay, T toggle on/off, Esc đa cấp, Ctrl+Z/Y, Alt-drag copy. Grid mở rộng bằng click viền ghost. Export/Import JSON `pixel-agents-layout.json`. Save debounce 500 ms → `saveLayout` → server.

---

## 5. Asset pipeline & licensing

### 5.1 Cấu trúc asset (`webview-ui/public/assets/`, ~210 KB)

| Loại | File | Kích thước | Ghi chú |
|---|---|---|---|
| Characters | `characters/char_0..5.png` | 112×96 mỗi file | 7 frame × 16 px ngang; 3 hàng hướng (down, up, right) × 32 px; frame: walk1-3, type1-2, read1-2 |
| Floors | `floors/floor_0..8.png` | 16×16 grayscale | colorize HSBC |
| Walls | `walls/wall_0.png` | 64×128 | 16 piece autotile 16×32 |
| Carpets | `carpets/carpet_0..2.png` | 64×64 | 16 case marching squares |
| Furniture | `furniture/<ID>/manifest.json + *.png` | 16×16 … 48×32, 16×64 | 25 loại |
| Pets | `pets/<id>/manifest.json + pet.png` | 96×96 | `{id, name}`; frame walk/idle × 3 hướng × 3 frame |
| Font | `fonts/FSPixelSansUnicode-Regular.ttf` | | pixel font UI |

### 5.2 Manifest furniture (ví dụ `furniture/PC/manifest.json`)

```json
{ "id": "PC", "name": "PC", "category": "electronics", "type": "group",
  "groupType": "rotation", "rotationScheme": "3-way-mirror",
  "canPlaceOnWalls": false, "canPlaceOnSurfaces": true, "backgroundTiles": 1,
  "members": [
    { "type": "group", "groupType": "state", "orientation": "front", "members": [
        { "type": "group", "groupType": "animation", "state": "on", "members": [
            { "type": "asset", "id": "PC_FRONT_ON_1", "file": "PC_FRONT_ON_1.png", "width": 16, "height": 32, "footprintW": 1, "footprintH": 2, "frame": 0 }, ... ] },
        { "type": "asset", "id": "PC_FRONT_OFF", "file": "PC_FRONT_OFF.png", ..., "state": "off" } ] },
    { "type": "asset", "id": "PC_BACK", ..., "orientation": "back" },
    { "type": "asset", "id": "PC_SIDE", ..., "orientation": "side", "mirrorSide": true } ] }
```

Group lồng nhau: rotation (`2-way`, `3-way-mirror`, `4-way`) → state (`on/off`) → animation (`frame`). `core/src/assets/build.ts buildFurnitureCatalog()` flatten thành `CatalogEntry[]` (`groupId`, `orientation`, `state`, `animationGroup`, `frame`, `mirrorSide`). `docs/external-assets.md` mô tả format để user thêm pack ngoài qua `Settings → Add Asset Directory` (lưu `externalAssetDirectories` trong config). `scripts/asset-manager.html` là editor manifest trực quan; `scripts/wall-tile-editor.html`, `scripts/jsonl-viewer.html` hỗ trợ.

### 5.3 Licensing — điểm cần lưu ý

- `LICENSE` (MIT, Pablo De Lucca 2026) bao phủ "the Software"; **không có file license/credits riêng cho art**.
- `README.md:48`: "6 diverse characters. These are based on the amazing work of JIK-A-4, Metro City (https://jik-a-4.itch.io/metrocity-free-topdown-character-pack)". Pack "free" trên itch.io thường có điều khoản riêng (có thể cấm redistribute/resale asset gốc hoặc yêu cầu credit) — **phải kiểm tra trang itch.io trước khi dùng nhân vật này trong sản phẩm thương mại**.
- `docs/external-assets.md:181` gợi ý pack "Office Interior Tileset (16x16)" của Donarg (itch.io) cho người dùng tự thêm — không bundle.
- Furniture/floor/wall/carpet/pet trong repo không có credit tác giả thứ ba → suy đoán do maintainer vẽ (nằm dưới MIT) nhưng không có tuyên bố rõ. Khuyến nghị: dùng format manifest + pipeline, nhưng **thay bộ sprite** bằng asset tự vẽ hoặc CC0 (vd. Kenney) để sạch pháp lý, hoặc liên hệ xác nhận.

---

## 6. State persistence

```
~/.pixel-agents/
  config.json            { vscode: AdapterSettings, standalone: AdapterSettings,
                           externalAssetDirectories[], hooksConsent{provider: granted|declined},
                           hooksEnabled{provider: bool}, directories[{name,path}] }
  vscode-state.json      { agents: PersistedAgent[], seats: {agentId: {palette, hueShift, seatId}} }
  standalone-state.json  (như trên, namespace riêng)
  layout.json            OfficeLayout — dùng chung giữa VS Code và standalone
  servers/<pid>.json     { port, pid, token, debugLog? } registry đa server
  server.json            legacy single target
  standalone-token       bearer token 0600
  hooks/claude-hook.js   script hook đã bundle
```

- `core/src/adapter.ts StateAdapter { loadAgents, saveAgents, loadSeats, saveSeats, getSetting, setSetting }`; implement `server/src/fileStateAdapter.ts` (atomic tmp+rename).
- `PersistedAgent` (`core/src/schemas.ts`): `{ id, sessionId?, terminalName, isExternal?, jsonlFile, projectDir, directoryName?, teamName?, agentName?, isTeamLead?, leadAgentId?, backgroundAgentToolIds?, palette?, hueShift? }`. Sub-agent không persist; teammate derived được tái tạo bởi scan 1 s.
- `AdapterSettings`: `soundEnabled, lastSeenVersion, alwaysShowLabels, ghostHeadlessAgents, watchAllSessions, hooksInfoShown, showAreas, areaMappings{dir: areaLabel[]}, bypassPermissions, showHostDirectories`.
- Restore: `existingAgents` message, webview buffer cho đến `layoutLoaded` (cả hai thứ tự đều xử lý, `existingAgents.ts`), `skipSpawnEffect: true`.
- Không có DB; không multi-user; mọi thứ là file JSON trên máy người dùng.

---

## 7. Hiệu năng

- **Không có DOM per-entity**: toàn bộ world trong 1 canvas, chỉ label/bubble là DOM (số lượng = số agent).
- **Sprite cache theo zoom**: mỗi SpriteData rasterize một lần/zoom vào offscreen canvas; `drawImage` nguyên khối; colorize cache theo key `(pattern,h,s,b,c)` / `"palette:hueShift"`.
- **Pixel-perfect**: zoom nguyên, toạ độ làm tròn, `imageSmoothingEnabled=false`.
- **Z-sort mỗi frame** O(n log n) với n = furniture + wall + characters + pets (vài trăm) — chấp nhận được; map 64×64 = 4096 tile vẽ mỗi frame không batch (không dirty-rect) → có thể là điểm nghẽn ở zoom cao/màn hình lớn; Pixel Agents không cache layer tile tĩnh.
- **Delta-time cap 0.1 s** chống nhảy khi tab ẩn.
- **Server**: polling 500 ms × số agent + scanner 1 s/3 s/30 s; đọc file bằng offset nên O(bytes mới). Delay 300 ms tool-done để tránh React batching. Hook body giới hạn 64 KB; buffer event 5 s cho session chưa đăng ký.
- **Transport**: broadcast mọi thay đổi tới mọi client, không diff; WS reconnect backoff + send queue.
- Bài học ghi trong CLAUDE.md: `fs.watch` không tin cậy trên Windows/macOS/WSL2; `--output-format stream-json` cần non-TTY stdin nên không dùng với terminal VS Code; hook IPC qua env var thất bại → chuyển sang HTTP + file discovery.

---

## 8. Đánh giá tái sử dụng cho AI WORKSPACE (NestJS + React, Phaser/Canvas)

### 8.1 Tái sử dụng trực tiếp (copy/adapt, giữ MIT attribution)

1. **Claude hook script + installer**: `claude-hook.ts` (fan-out POST, registry PID) và logic an toàn của `claudeHookInstaller.ts` (consent, backup, atomic, hook identity). Chỉ cần đổi đường dẫn `~/.pixel-agents` → `~/.ai-workspace` và endpoint.
2. **`normalizeHookEvent` + `formatToolStatus`** trong `claude.ts`; bảng `CLAUDE_HOOK_EVENTS`; `contextWindowForModel`.
3. **Transcript tail reader** (`fileWatcher.ts readNewLines`, `startFileWatching`) + các nhánh record trong `transcriptParser.ts` (`assistant/tool_use`, `user/tool_result`, `system/turn_duration`, `progress`, `usage`), và `normalizeProjectPath.ts`.
4. **Engine webview**: `tileMap.ts` (BFS), `characters.ts` (FSM), `gameLoop.ts`, `spriteCache.ts`, `colorize.ts`, `wallTiles.ts`, `carpetTiles.ts`, `projection.ts`, `seatPlacement.ts`, `layoutSerializer.ts` (seats/blocked tiles). Nếu dùng Canvas: dùng luôn `renderer.ts`. Nếu dùng Phaser: giữ FSM/pathfinding/layout, thay renderer bằng Phaser Sprite + `setDepth(zY)`.
5. **Asset format**: manifest furniture (rotation/state/animation), character sheet 112×96, wall autotile 4-bit, carpet marching squares, `pngDecoder.ts`, `build.ts buildFurnitureCatalog`.
6. **Transport interface** `MessageTransport` + `webSocketTransport.ts` (backoff, queue).
7. **Layout editor logic thuần** (`editorActions.ts`, `editorState.ts`, undo stroke session) — UI React phải viết lại theo design system của ta.
8. **E2E pattern**: `e2e/fixtures/mock-claude-runner.cjs` (giả lập Claude ghi JSONL + bắn hook theo scenario) rất hữu ích để test provider không cần Claude thật.

### 8.2 Tham khảo (reference-only)

- `AgentRuntime`, `AgentStateStore`, `SessionRouter`, `DismissalTracker`, `HookEventHandler`: thiết kế tốt nhưng gắn chặt single-process, global module state (`setHookProvider`, `setTeamProvider`…), và nhiều nhánh team Claude phức tạp. Với NestJS ta nên viết service tương đương (per-workspace store, EventEmitter2/RxJS), lấy ý tưởng: `hookDelivered` gate timer, 300 ms tool-done delay, buffer event trước khi agent đăng ký, sentinel `/clear`.
- `claudeTeamProvider.ts`, `subagentWatch.ts`, `docs/research/claude-harness-spawn-formats.md`: tài liệu quý về format spawn sub-agent/teammate của Claude Code 2.1.x — dùng để thiết kế theo dõi sub-agent, nhưng domain Team/Task của ta khác (company/role/task board).
- `ptySessionManager.ts` + xterm mirror: tham khảo để nhúng terminal trong web app của ta.
- `consentGate/consentExecutor`, ADR 0001: mô hình consent khi ghi vào settings của tool bên thứ ba.
- UI React (`App.tsx`, `SettingsModal`, `MobileShell`, `AgentCard`): tham khảo UX, không copy (ESLint rule pixel-aesthetic, VS Code-specific).

### 8.3 Phải viết mới

- **Codex CLI provider** (stream/JSON-RPC, không hook) và `StreamProvider` kind; chuẩn hoá `AgentEvent` mở rộng (thinking, message, tokenUsage, error, cost).
- **Multi-agent company domain**: role, task, assignment, inter-agent messaging — Pixel Agents chỉ có session → character 1:1.
- **Backend NestJS**: WebSocket gateway, REST, DB (Postgres/SQLite) thay cho file JSON; multi-workspace/multi-user; auth.
- **Nhân vật chibi + asset riêng**: sprite sheet mới theo layout 7-frame×3-dir hoặc định nghĩa lại; thêm state `thinking`, `talking`, `error`, `celebrate` (Pixel Agents chỉ có walk/type/read).
- **Phaser integration** (nếu chọn): scene, tilemap layer từ `OfficeLayout`, depth sort, camera; hoặc giữ Canvas và chỉ bọc lại renderer.
- **Layout editor UI** theo design system riêng; tính năng room/zone cho team.
- **Thinking detection**: hiện không có tín hiệu "thinking"; có thể suy từ record `assistant` có block `thinking` trong JSONL (parser hiện bỏ qua) hoặc khoảng giữa `UserPromptSubmit` và `PreToolUse` đầu tiên — cần thêm.

### 8.4 Rủi ro & lưu ý

- License art nhân vật (JIK-A-4) chưa rõ → thay asset.
- Format JSONL/hook của Claude Code thay đổi theo version (provider tương thích tới 2.1.220) → cần test fixture + version guard.
- Windows: `fs.watch` không tin cậy, drive letter case → giữ polling + case-insensitive match (đã có trong `getSessionDirs`).
- Hook script chạy `node` mỗi event (~12 event/turn) → nhẹ nhưng cần timeout ngắn và không bao giờ block Claude.
- Security: hook endpoint chỉ bind 127.0.0.1 + Bearer; nếu backend ta chạy trong Docker/remote cần cơ chế relay khác (hook script của máy dev → backend).

---

## 9. Bảng tổng hợp module

| Module | Path | Tái sử dụng trực tiếp | Cần chỉnh sửa | Viết mới | Ghi chú |
|---|---|---|---|---|---|
| AgentEvent + HookProvider interface | `core/src/provider.ts` | ✔ | Thêm kind `thinking/message/usage/error`, bỏ sentinel `'current'` | | Nền tảng Provider Adapter |
| TeamProvider interface | `core/src/teamProvider.ts` | | ✔ (map sang domain Team/Task của ta) | | Claude-centric |
| MessageTransport + WS transport | `core/src/transport.ts`, `webview-ui/src/transport/webSocketTransport.ts` | ✔ | Đổi message types | | Backoff + queue tốt |
| AsyncAPI contract + codegen | `core/asyncapi.yaml`, `scripts/generate-messages.ts` | | ✔ (viết contract riêng, giữ quy trình) | | Hoặc dùng zod/ts-rest |
| Claude normalizeHookEvent, formatToolStatus | `server/src/providers/hook/claude/claude.ts` | ✔ | Đổi id/path | | Cốt lõi quan sát Claude |
| Claude hook script | `server/src/providers/hook/claude/hooks/claude-hook.ts` | ✔ | Đổi registry dir, endpoint | | Fan-out nhiều server |
| Hook installer (settings.json) | `server/src/providers/hook/claude/claudeHookInstaller.ts` | ✔ | Đổi path/tên | | Rất an toàn, nhiều test |
| Consent gate/executor | `server/src/providers/hook/consent*.ts` | | ✔ | | Logic UX consent |
| Transcript parser (JSONL) | `server/src/transcriptParser.ts` | ✔ (nhánh record cơ bản) | Tách khỏi global state, thêm `thinking` | | Claude-specific là hợp lý |
| File watcher / tail | `server/src/fileWatcher.ts` | ✔ (`readNewLines`, `startFileWatching`) | Bỏ phần team/sidecar nếu chưa cần | | Poll 500 ms, lineBuffer |
| Context usage | `server/src/contextUsage.ts` | ✔ | | | Snapshot, không cộng dồn |
| AgentRuntime / StateStore / SessionRouter | `server/src/agentRuntime.ts`, `agentStateStore.ts`, `sessionRouter.ts` | | | ✔ (NestJS service, per-workspace) | Tham khảo ý tưởng gate timer, buffer |
| Timer manager | `server/src/timerManager.ts` | | ✔ | | 7 s permission, 5 s text-idle |
| Fastify HTTP/WS + auth | `server/src/httpServer.ts`, `wsAuth.ts` | | | ✔ (NestJS gateway) | Tham khảo 2-tier auth |
| PTY session + xterm mirror | `server/src/terminal/*` | | ✔ | | Nếu cần terminal trong web |
| Standalone launcher | `server/src/terminal/standaloneAgentLauncher.ts` | | ✔ | | `claude --session-id <uuid>` |
| Persistence (file JSON) | `server/src/fileStateAdapter.ts`, `layoutPersistence.ts`, `configPersistence.ts` | | | ✔ (DB) | Giữ atomic write cho layout export |
| Office types / layout model | `webview-ui/src/office/types.ts` | ✔ | Thêm room/zone, agent role | | `OfficeLayout` JSON tốt |
| BFS pathfinding | `webview-ui/src/office/layout/tileMap.ts` | ✔ | | | Có thể nâng A* nếu map lớn |
| Character FSM + wander | `webview-ui/src/office/engine/characters.ts` | ✔ | Thêm state thinking/talking/error | | Độc lập renderer |
| Seat derivation / assignment | `layoutSerializer.ts`, `engine/seatPlacement.ts`, `officeState.ts findFreeSeat` | ✔ | Area → team/room | | |
| OfficeState (world) | `webview-ui/src/office/engine/officeState.ts` | | ✔ (giữ API, bỏ greeter/consent) | | 1229 dòng, imperative |
| Game loop | `engine/gameLoop.ts` | ✔ | | | Nếu Canvas; Phaser có loop riêng |
| Renderer Canvas 2D | `engine/renderer.ts` | ✔ (Canvas) | | ✔ (Phaser) | Z-sort rule tái dùng cho `setDepth` |
| Sprite cache / outline | `sprites/spriteCache.ts` | ✔ (Canvas) | | | Phaser dùng texture atlas |
| Colorize / hue shift | `office/colorize.ts` | ✔ | | | Có thể chạy ở build-time |
| Wall autotile, carpet marching squares | `wallTiles.ts`, `sprites/carpetTiles.ts` | ✔ | | | Bitmask conventions |
| Floor tiles | `office/floorTiles.ts` | ✔ | | | |
| Matrix spawn effect | `engine/matrixEffect*.ts` | ✔ | | | Tuỳ chọn |
| Pets | `engine/petEntity.ts`, `sprites/petSpriteData.ts` | ✔ | | | Tuỳ chọn |
| ToolOverlay (label/bubble DOM) | `office/components/ToolOverlay.tsx`, `projection.ts` | | ✔ | | Thay bằng UI của ta |
| OfficeCanvas (input, zoom, DPR) | `office/components/OfficeCanvas.tsx` | | ✔ | | Tách pan/zoom/hit-test |
| Editor logic thuần | `office/editor/editorActions.ts`, `editorState.ts`, `editorRenderState.ts` | ✔ | | | Undo stroke session |
| Editor toolbar UI | `office/editor/EditorToolbar.tsx`, `components/*` | | | ✔ | Theo design system riêng |
| Message → OfficeState mapping | `hooks/useExtensionMessages.ts` | | ✔ (viết lại gọn theo protocol mới) | | 925 dòng, nhiều nhánh team |
| Asset decoders / catalog build | `core/src/assets/*`, `server/src/assetLoader.ts` | ✔ | Bỏ phụ thuộc `vscode.Webview` | | pngjs |
| Furniture manifest format | `docs/external-assets.md`, `webview-ui/public/assets/furniture/*/manifest.json` | ✔ (format) | | | Giữ tương thích để nhận pack cộng đồng |
| Sprite assets (characters) | `webview-ui/public/assets/characters/*.png` | | | ✔ (vẽ lại chibi) | License JIK-A-4 chưa xác minh |
| Sprite assets (furniture/floor/wall/carpet/pet) | `webview-ui/public/assets/**` | ✔ (tạm, MIT suy đoán) | | | Nên thay dần bằng asset riêng |
| Asset tools | `scripts/asset-manager.html`, `wall-tile-editor.html`, `jsonl-viewer.html` | ✔ | | | Công cụ nội bộ |
| Mock Claude e2e runner | `e2e/fixtures/mock-claude-runner.cjs`, `e2e/helpers/mock-claude.ts` | ✔ | Đổi đường dẫn | | Test provider không cần Claude thật |
| VS Code adapter | `adapters/vscode/*` | | | ✔ (không cần, ta là web app) | Chỉ tham khảo launch |
| Codex / Gemini provider | — | | | ✔ | Chưa tồn tại trong Pixel Agents |
| Domain company/role/task | — | | | ✔ | Ngoài phạm vi Pixel Agents |
