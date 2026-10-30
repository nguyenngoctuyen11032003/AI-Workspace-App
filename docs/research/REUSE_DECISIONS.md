# REUSE_DECISIONS — Bảng quyết định tái sử dụng (master table)

> Tác giả: Technical Research Lead (tổng hợp Phase 0). Ngày: 2026-10-30.
> Nguồn: các bảng "Tổng hợp" của `CHATDEV_ANALYSIS.md` §12, `PIXEL_AGENTS_ANALYSIS.md` §9, `MY_VIRTUAL_OFFICE_ANALYSIS.md` §11.3, `OPENHANDS_ANALYSIS.md` §12; `LICENSE_AND_REUSE.md` §3–4; `DECISION_LOG.md` (D-001…D-016).
> Đây là tài liệu kỹ thuật, không phải tư vấn pháp lý.
> **[OWR-2 Q-15 / D-012]** Asset mặc định chỉ CC0; font OFL được phép nếu lưu license/NOTICE; CC-BY chỉ khi Owner phê duyệt từng package; asset MIT nguồn gốc không rõ (ví dụ Pixel Agents furniture/tile, RD-033) không bundle mặc định, chặt hơn các dòng "ALLOW-COND/Dùng tạm" bên dưới. Tái dùng code Apache-2.0/MIT sau khi xác minh license và attribution không đổi. My Virtual Office (AGPL) chỉ tham khảo ý tưởng; không sao chép code, prompt, layout, asset.

---

## 1. Mục đích

Một bảng duy nhất cho mọi module "tái sử dụng / tham khảo / viết mới", mỗi dòng có: nguồn, mức tái sử dụng, license verdict, yêu cầu attribution, ID quyết định và nhãn bằng chứng. Mục tiêu là để (a) không ai copy nhầm code AGPL, (b) mọi file có nguồn gốc bên thứ ba đều có attribution đúng, (c) Phase 1 biết chính xác phần nào cần viết.

## 2. Phạm vi

- 4 repo: ChatDev (Apache-2.0), Pixel Agents (MIT), OpenHands/Agent Canvas (MIT), My Virtual Office (AGPL-3.0, chỉ ý tưởng).
- Bao gồm cả các module **phải viết mới** vì không repo nào có.
- Không bao gồm dependency npm thông thường (xử lý bằng công cụ quét license trong CI, xem §7).

## 3. Quy ước đọc bảng

| Ký hiệu | Ý nghĩa |
|---|---|
| **Tái sử dụng** | Copy/port nội dung hoặc mã gần nguyên trạng. Bắt buộc có attribution theo license. |
| **Chỉnh sửa** | Dùng làm nền nhưng phải đổi đáng kể (đường dẫn, tên, interface, nội dung). Vẫn là derivative work. |
| **Viết mới** | Không copy. Có thể tham khảo ý tưởng/thiết kế; mã tự viết. |
| `ALLOW` / `ALLOW-COND` / `REF-ONLY` / `DENY` | Verdict license: cho phép / cho phép có điều kiện / chỉ tham khảo ý tưởng / cấm copy. |
| **Đã xác minh** | Có bằng chứng đã đọc mã/license/chạy lệnh trong Phase 0 (không có nghĩa là đã được luật sư duyệt). |
| **Đề xuất** | Khuyến nghị của nhóm nghiên cứu, chưa kiểm chứng bằng chạy thử hoặc chưa có phê duyệt. |

Decision ID: `RD-xxx` là quyết định tái sử dụng của tài liệu này; cột cuối ghi thêm `D-0xx` của `DECISION_LOG.md` khi liên quan. Mọi RD đều ở trạng thái **Đề xuất / Chờ duyệt**.

Các cột đầu theo đúng yêu cầu; hai cột cuối (Attribution, Nhãn) bổ sung để mỗi dòng tự đủ thông tin pháp lý.

---

## 4. Master table

### 4.1 ChatDev (Apache-2.0)

Attribution Apache-2.0 §4: giữ bản sao license, ghi thông báo thay đổi trong file đã sửa, giữ copyright/NOTICE (ChatDev hiện **không có file NOTICE** — đã kiểm tra), không dùng tên/logo/trademark.

| Module | Repository | Có thể tái sử dụng | Cần chỉnh sửa | Cần viết mới | License verdict | Decision ID | Attribution | Nhãn |
|---|---|---|---|---|---|---|---|---|
| Role prompts 1.0 (9 vai: CEO, CPO, CTO, CHRO, Counselor, Programmer, Reviewer, Tester, CCO) — `v1.1.6:CompanyConfig/Default/RoleConfig.json` | ChatDev | Nội dung prompt | Đổi placeholder `{task}`; bỏ yêu cầu định dạng code-block (agent của ta có tool thật); chuyển thành seed data (D-016) | | ALLOW | RD-001 / D-012, D-016 | Có: header + mục ChatDev trong THIRD_PARTY_NOTICES + ghi thay đổi | Đã xác minh (nguồn tag), Đề xuất (cách chỉnh) |
| Phase prompts 1.0 (13 phase) — `PhaseConfig.json`, điều kiện dừng `<INFO>` | ChatDev | Nội dung prompt | Thay "Source Codes: {codes}" bằng chỉ dẫn dùng tool đọc workspace | | ALLOW | RD-002 | Như RD-001 | Đã xác minh / Đề xuất |
| Chain config (Default/Art/Human/Incremental) — `ChatChainConfig.json` | ChatDev | Cấu trúc chuỗi | Map `ComposedPhase`/`cycleNum` sang loop node | | ALLOW | RD-003 | Như RD-001 | Đề xuất |
| `background_prompt` / `${COMMON_PROMPT}` | ChatDev | Văn bản | Rút gọn theo tone công ty AI WORKSPACE | | ALLOW | RD-001 | Như RD-001 | Đề xuất |
| Workflow mẫu `yaml_instance/ChatDev_v1.yaml` (1021 dòng) | ChatDev | File YAML (làm fixture import) | Đổi tên tool/provider sang adapter của ta | | ALLOW | RD-004 | Header YAML ghi nguồn + thay đổi | Đã xác minh (file tồn tại), Đề xuất (cách dùng) |
| YAML schema (`yaml_template/design.yaml`, `entity/configs/**`): `nodes/edges/start/end/vars`, edge `trigger/condition/carry_data/keep_message/clear_context/process/dynamic` | ChatDev | **Tên field và ngữ nghĩa** | | Zod/TS types + validator 3 lớp tự viết | ALLOW (field name là interface; mã Python không copy) | RD-005 / D-003 | Ghi "schema tương thích với ChatDev 2.0" trong docs; header file Zod nếu dịch sát | Đã xác minh (nguồn), Đề xuất (tương thích) |
| Event enums: `EventType` (NODE_START/END, EDGE_PROCESS, MODEL_CALL, TOOL_CALL, AGENT_CALL, HUMAN_INTERACTION, THINKING_PROCESS, MEMORY_OPERATION, WORKFLOW_START/END) | ChatDev | Danh sách tên và ý nghĩa | Đổi sang quy ước event của ta (D-014), thêm id/seq | Bảng `agent_events` + producer | ALLOW | RD-006 / D-014 | Chú thích nguồn ở file enum | Đã xác minh / Đề xuất |
| WS protocol (`connection`, `log`, `human_input_required`, `artifact_created`, `workflow_completed`, `session_resumed`…) | ChatDev | Danh sách message type | Socket.IO gateway tự viết (D-014) | NestJS Gateway | ALLOW | RD-007 | Tài liệu thiết kế ghi nguồn | Đề xuất |
| SKILL.md format (`name/description/allowed-tools`) | ChatDev | Chuẩn mở (trùng Agent Skills của Claude Code) | Mở rộng `skill.yaml` + `metadata.aiw-*` (D-013) | Loader TS nếu cần | ALLOW | RD-008 / D-013 | Không bắt buộc (chuẩn mở); ghi nếu copy 3 skill mẫu | Đã xác minh |
| 3 skill mẫu `.agents/skills/**` | ChatDev | Nội dung | Kiểm tra từng file có license riêng không | | ALLOW-COND | RD-008 | Như RD-001 nếu dùng | Đề xuất |
| Test bảo mật (`tests/test_session_workspace_security.py`, `test_attachment_boundary_security.py`, `test_code_execution_security.py`) | ChatDev | Ca kiểm thử (ý tưởng + case) | Port sang test runner của ta | Viết lại bằng TS | ALLOW | RD-009 | Header ghi nguồn trong file test port | Đề xuất |
| Sprite pixel-art 12 nhân vật `frontend/public/sprites/{1..12}-{D,L,R,U}-{1..3}.png` | ChatDev | **Không dùng cho tới khi xác minh** | | Vẽ asset riêng (RD-034) | ALLOW-COND (Apache áp dụng cho repo nhưng chưa tìm thấy tuyên bố nguồn gốc art) | RD-010 / D-012 | Nếu dùng: Apache + ghi thay đổi | **Đề xuất — chưa xác minh nguồn gốc** |
| `GraphExecutor`, strategies DAG/Cycle/MajorityVote, `ParallelExecutor`, `ResourceManager` | ChatDev | | | "Conductor" (D-003) viết mới trên BullMQ + Postgres | REF-ONLY (thuật toán, không copy Python) | RD-011 / D-003 | Không bắt buộc cho ý tưởng; ghi credit tự nguyện | Đã xác minh (đọc mã), Đề xuất |
| Cycle engine (Tarjan SCC + super-node + initial node) | ChatDev | Thuật toán | | Port sang TS | REF-ONLY | RD-011 | Như trên | Đề xuất |
| Dynamic Map/Tree edge (scatter-gather) | ChatDev | | | Viết mới | REF-ONLY | RD-011 | Như trên | Đề xuất |
| Human-in-loop (`HumanPromptService`, `PromptChannel`, timeout 1800 s) | ChatDev | Protocol + timeout | Pending prompt phải persist DB (ChatDev dùng `Future` RAM) | Approval `WAITING_HUMAN` (D-003) | REF-ONLY | RD-012 | Như trên | Đã xác minh |
| `WorkspaceArtifactHook` (sha256 snapshot trước/sau node) | ChatDev | Ý tưởng | | Watcher/`git status` trong worktree | REF-ONLY | RD-013 | Không | Đề xuất |
| `TokenTracker` (per node/model/provider) | ChatDev | Data model | | Bảng `token_usage` | REF-ONLY | RD-014 | Không | Đề xuất |
| Provider layer OpenAI/Gemini, `ToolManager` function-calling, Python node | ChatDev | | | `AgentBrainAdapter`; Docker sandbox | Không dùng (khác ngôn ngữ, không sandbox) | RD-015 / D-004, D-005, D-007 | Không | Đã xác minh |
| `WorkflowSessionStore` in-memory, `WareHouse/` | ChatDev | | | PostgreSQL (D-002) | Không dùng | RD-015 | Không | Đã xác minh |
| Frontend Vue 3 + Vue Flow | ChatDev | Chỉ UX (form sinh từ schema) | | React Flow builder | REF-ONLY | RD-016 | Không | Đề xuất |

### 4.2 Pixel Agents (MIT)

Attribution MIT: giữ dòng `Copyright (c) 2026 Pablo De Lucca` và nội dung license MIT trong THIRD_PARTY_NOTICES; header trong mỗi file copy/port. Chỉ lấy từ commit `add749a` trở về sau; tuyệt đối không lấy PNG từ lịch sử git cũ (tileset proprietary, theo `LICENSE_AND_REUSE.md` §6.2).

| Module | Repository | Có thể tái sử dụng | Cần chỉnh sửa | Cần viết mới | License verdict | Decision ID | Attribution | Nhãn |
|---|---|---|---|---|---|---|---|---|
| Provider interface + `AgentEvent` — `core/src/provider.ts` (151 dòng, 10 kind) | Pixel Agents | Interface làm nền | Thêm kind `thinking/message/usage/error`; bỏ sentinel `toolId:'current'` bằng correlate id bắt buộc | Mở rộng thành `AgentBrainAdapter` | ALLOW | RD-017 / D-004 | MIT header + NOTICES | Đã xác minh (đọc mã), Đề xuất (mở rộng) |
| `claude.ts`: `normalizeHookEvent`, `formatToolStatus`, `CLAUDE_HOOK_EVENTS`, `contextWindowForModel` | Pixel Agents | Mã | Đổi id/đường dẫn | | ALLOW | RD-018 | MIT header | Đã xác minh |
| Hook script `claude-hook.ts` (fan-out POST, registry PID) | Pixel Agents | Mã | Đổi `~/.pixel-agents` → `~/.ai-workspace`, endpoint | | ALLOW | RD-019 | MIT header | Đã xác minh |
| Hook installer `claudeHookInstaller.ts` (consent, backup, atomic write, hook identity) | Pixel Agents | Logic an toàn | Đổi path/tên | | ALLOW | RD-020 / D-008 | MIT header | Đã xác minh (nhiều test theo tài liệu) |
| Consent gate/executor | Pixel Agents | Mô hình UX | Viết theo design system của ta | UI consent | ALLOW | RD-020 | MIT nếu copy logic | Đề xuất |
| Transcript tail (`fileWatcher.ts readNewLines`, `startFileWatching`, poll 500 ms), `transcriptParser.ts` (các nhánh record cơ bản), `normalizeProjectPath.ts`, `contextUsage.ts` | Pixel Agents | Mã | Tách khỏi global state; thêm nhánh `thinking`; bỏ phần team/sidecar nếu chưa cần | Version guard theo Claude Code version | ALLOW | RD-021 | MIT header | Đã xác minh |
| `AgentRuntime`, `AgentStateStore`, `SessionRouter`, `DismissalTracker`, `HookEventHandler` | Pixel Agents | Ý tưởng (hookDelivered gate, tool-done delay 300 ms, buffer trước khi đăng ký) | | NestJS service per-workspace (EventEmitter2/RxJS) | REF-ONLY (global state, single-process) | RD-022 | Không bắt buộc | Đề xuất |
| Fastify HTTP/WS + auth 2 tầng | Pixel Agents | Ý tưởng auth | | NestJS Gateway | REF-ONLY | RD-022 | Không | Đề xuất |
| `MessageTransport` + `webSocketTransport.ts` (backoff, queue) | Pixel Agents | Mã | Đổi message type; cân nhắc Socket.IO (D-014) | | ALLOW | RD-023 | MIT header | Đề xuất |
| AsyncAPI contract + codegen | Pixel Agents | Quy trình | | Contract riêng (Zod/ts-rest) | ALLOW (quy trình) | RD-023 | Không | Đề xuất |
| Tile map + **BFS 4 hướng** — `office/layout/tileMap.ts` (105 dòng) | Pixel Agents | Mã | Có thể nâng A* nếu map lớn | | ALLOW | RD-024 / D-011 | MIT header | Đã xác minh |
| Character **FSM** (`idle/walk/type`, wander) — `engine/characters.ts` (342 dòng) | Pixel Agents | Mã (độc lập renderer) | Thêm state `thinking/talking/error/celebrate`, ánh xạ từ event thật | | ALLOW | RD-025 / D-011 | MIT header | Đã xác minh |
| Seat derivation/assignment (`layoutSerializer.ts`, `seatPlacement.ts`, `findFreeSeat`) | Pixel Agents | Mã | Area → team/room | | ALLOW | RD-026 | MIT header | Đã xác minh |
| `OfficeLayout` types + serializer (JSON layout) | Pixel Agents | Mô hình | Thêm room/zone, role | | ALLOW | RD-026 | MIT header | Đã xác minh |
| `OfficeState` (1229 dòng, imperative) | Pixel Agents | Giữ API | Bỏ greeter/consent | | ALLOW-COND (cần chỉnh nhiều) | RD-026 | MIT header | Đề xuất |
| Game loop `gameLoop.ts` | Pixel Agents | Mã nếu dùng Canvas | Phaser có loop riêng → không dùng | | ALLOW | RD-027 | MIT header | Đề xuất |
| Renderer `renderer.ts` (1014 dòng), `spriteCache.ts`, outline | Pixel Agents | Chỉ **quy tắc z-sort** | D-011 chọn Phaser: thay bằng Sprite + `setDepth(zY)` + texture atlas | Phaser scene | ALLOW (quy tắc); mã không dùng | RD-027 / D-011 | MIT nếu copy | Đã xác minh (mâu thuẫn C-4 trong ARCHITECTURE_COMPARISON) |
| `colorize.ts`, `wallTiles.ts` (autotile 4-bit), `carpetTiles.ts`, `floorTiles.ts`, `projection.ts` | Pixel Agents | Mã/bitmask | Chạy build-time nếu cần | | ALLOW | RD-028 | MIT header | Đã xác minh |
| Matrix spawn effect, pets (`petEntity.ts`) | Pixel Agents | Mã | Tùy chọn | | ALLOW | RD-028 | MIT header | Đề xuất |
| **Layout editor** logic thuần (`editorActions.ts`, `editorState.ts`, undo stroke session 50 bước) | Pixel Agents | Mã | Tách khỏi UI | | ALLOW | RD-029 | MIT header | Đã xác minh |
| Layout editor UI (`EditorToolbar.tsx`…) | Pixel Agents | Chỉ UX | | React theo design system (D-015) | REF-ONLY | RD-029 | Không | Đề xuất |
| ToolOverlay, OfficeCanvas, `useExtensionMessages.ts` (925 dòng) | Pixel Agents | Ý tưởng | Viết lại gọn | | REF-ONLY | RD-030 | Không | Đề xuất |
| Asset format: furniture `manifest.json` (rotation→state→animation), character sheet 112×96, `build.ts buildFurnitureCatalog`, `pngDecoder.ts` | Pixel Agents | **Format + mã** | Bỏ phụ thuộc `vscode.Webview` | | ALLOW | RD-031 | MIT header | Đã xác minh |
| `docs/external-assets.md` + cơ chế thư mục asset ngoài | Pixel Agents | Mô hình BYO assets | | Wizard import trong app | ALLOW | RD-031 | MIT | Đề xuất |
| E2E: `e2e/fixtures/mock-claude-runner.cjs`, `e2e/helpers/mock-claude.ts` | Pixel Agents | Mã | Đổi đường dẫn | | ALLOW | RD-032 | MIT header | Đã xác minh |
| Asset tools `scripts/asset-manager.html`, `wall-tile-editor.html`, `jsonl-viewer.html` | Pixel Agents | Công cụ nội bộ | | | ALLOW | RD-032 | MIT | Đề xuất |
| **Sprite nhân vật** `characters/char_0..5.png` (dựa trên JIK-A-4 MetroCity) | Pixel Agents | Chỉ sau khi xác minh | | Vẽ chibi riêng (RD-034) | ALLOW-COND | RD-033 | Nếu CC0 xác nhận: ghi credit tự nguyện; nếu không: không dùng | **Đề xuất — mâu thuẫn C-2** (README chỉ credit, LICENSE_AND_REUSE nói CC0 từ trang itch.io) |
| Sprite furniture/floor/wall/carpet/pet (post-#117) | Pixel Agents | ~~Dùng tạm~~ **Không bundle mặc định (OWR-2 Q-15)** | Thay bằng asset CC0/tự vẽ | | ~~ALLOW-COND~~ DENY mặc định cho tới khi tác giả xác nhận nguồn gốc và Owner duyệt (MIT theo repo, không có tuyên bố art riêng) | RD-033 | MIT + `CREDITS.json` | Đề xuất; chưa có xác nhận bằng văn bản của tác giả |
| Font `FSPixelSansUnicode-Regular.ttf` | Pixel Agents | **Không bundle** | | Thay bằng Press Start 2P/Pixelify Sans (OFL) | DENY cho tới khi xác minh (license mâu thuẫn OFL vs personal-use) | RD-035 | — | Đề xuất |
| `claudeTeamProvider.ts`, `subagentWatch.ts`, `docs/research/claude-harness-spawn-formats.md` | Pixel Agents | Tài liệu format spawn sub-agent (Claude Code 2.1.x) | | Domain team/task riêng | REF-ONLY | RD-022 | Không | Đã xác minh (tồn tại) |
| `ptySessionManager.ts` + xterm mirror | Pixel Agents | Ý tưởng terminal trong web | Nếu dùng: Windows cần xử lý PTY riêng | | REF-ONLY | RD-022 | Không | Đề xuất |
| Adapter VS Code | Pixel Agents | | | Không cần (ta là web app) | N/A | RD-036 | Không | Đã xác minh |

### 4.3 OpenHands / Agent Canvas (MIT)

Attribution MIT: `Copyright © 2025 OpenHands contributors`. Không dùng logo, tên thương mại, ảnh `public/*.png`. Phạm vi "OpenHands" ở đây là: (a) repo Agent Canvas đã clone, (b) Agent Server + client nằm ở repo khác. **Dòng (b) chưa được kiểm tra license.**

| Module | Repository | Có thể tái sử dụng | Cần chỉnh sửa | Cần viết mới | License verdict | Decision ID | Attribution | Nhãn |
|---|---|---|---|---|---|---|---|---|
| **Agent Server** (`openhands-agent-server==1.59.0`, `uvx`) chạy như service ngoài | `OpenHands/software-agent-sdk` (repo khác, không vendored) | Dùng như dịch vụ (không copy mã) | Version-gate qua `/server_info` | `OpenHandsProvider` trong NestJS | **Chưa xác minh license** (giả định MIT theo repo hiện tại, không có bằng chứng) | RD-037 / D-006 | Nếu phân phối image/binary: bắt buộc theo license thật | **Đề xuất — cần kiểm tra** |
| **Docker runtime** (`ghcr.io/openhands/agent-server:<ver>-python`, per-conversation Docker với giới hạn CPU/mem/pids) | software-agent-sdk | Dùng image như dependency runtime (Tier 2 sandbox) | Thêm tool/ngôn ngữ nếu cần | Docker sandbox Tier 1 tự viết (D-007) | Chưa xác minh (như trên) | RD-038 / D-007 | Theo license image | Đề xuất; chưa chạy thử (Docker daemon chưa chạy lúc kiểm tra) |
| **`@openhands/typescript-client`** (npm 1.59.0) | OpenHands/typescript-client | Gọi từ NestJS (`ConversationClient`, `FileClient`, `RemoteWorkspace`, …) | Chỉ wrapper mỏng | | ALLOW-COND (license package chưa đọc) | RD-039 / D-006 | Giữ trong `THIRD_PARTY_NOTICES.deps` | **Đề xuất — cần kiểm tra license npm** |
| **Event schema TS** `src/types/agent-server/core/**` (`BaseEvent`, `ActionEvent`, `ObservationEvent`, `MessageEvent`, `ConversationStateUpdateEvent`, `CondensationEvent`, `HookExecutionEvent`…) | OpenHands (Canvas) | Làm tài liệu/types tham chiếu | Map sang event chuẩn của ta (envelope + `kind`) | Event model riêng | ALLOW | RD-040 / D-014 | MIT header nếu copy file | Đã xác minh (đọc mã) |
| **Session frames** `session-frames.ts` (durable/transient/delta, `after_seq` resume) | OpenHands (Canvas) | Mô hình giao thức | | Replay bằng `lastEventId` (D-014) | ALLOW | RD-040 | MIT header nếu copy | Đã xác minh |
| **Client tools** (`canvas-ui-client-tool.ts`, `launch-child-conversation-client-tool.ts`, `request-secret-client-tool.ts`) | OpenHands (Canvas) | Spec JSON của tool | Đổi tên/mô tả theo UI của ta | Dispatcher mới | ALLOW | RD-041 | MIT header nếu copy spec | Đã xác minh |
| `agent-server-adapter.ts` (`buildStartConversationRequest`, confirmation policy, `LookupSecret`) | OpenHands (Canvas) | Cách lắp payload | Trích một phần | Trong `OpenHandsProvider` | ALLOW | RD-042 | MIT header | Đề xuất |
| WebSocket reconnect/batcher (`use-websocket.ts`, `conversation-websocket-context.tsx`) | OpenHands (Canvas) | Logic | Gắn Zustand → tách | Viết lại | REF-ONLY | RD-042 | Không | Đề xuất |
| `scripts/ingress.mjs`, `dev-process-utils.mjs`, `logger.mjs` | OpenHands (Canvas) | Mã | Windows: kill process tree | | ALLOW | RD-043 | MIT header | Đề xuất |
| Electron packaging (`electron/main.mjs`, `electron-builder.config.mjs`, `download-uv.mjs`, `download-node.mjs`) | OpenHands (Canvas) | **Ý tưởng** (shell + sidecar + bundle Node/uv + PATH) | Map sang Tauri sidecar (phase sau) | Tauri shell | REF-ONLY (Tauri không dùng mã Electron) | RD-044 / D-001 | Không bắt buộc | Đã xác minh |
| ACP provider registry (`acp-providers.ts`, `docs/ACP_AGENTS.md`) | OpenHands (Canvas) | Danh sách lệnh | Dùng khi chạy Claude Code/Codex làm host trong sandbox | | ALLOW | RD-045 | MIT header | Đề xuất |
| Skills catalog `@openhands/extensions`, `.agents/skills/` | OpenHands | Catalog SKILL.md | Kiểm tra license từng skill | | ALLOW-COND | RD-046 / D-013 | Theo từng skill | Đề xuất |
| Helm chart `helm/agent-canvas/` | OpenHands (Canvas) | Mẫu | Đổi image/env | | ALLOW | RD-047 | MIT header | Đề xuất (chưa cần ở MVP) |
| Config pattern `config/defaults.json` → `defaults.env` | OpenHands (Canvas) | Mẫu | | | ALLOW | RD-047 | Không bắt buộc | Đề xuất |
| Docs bảo mật `docs/SELF_HOSTING.md`, `DefenseClaw.md` | OpenHands (Canvas) | Checklist | | | ALLOW | RD-048 | Không | Đề xuất |
| UI React Canvas (`src/components/**`, `stores/**`, `routes/**`) | OpenHands (Canvas) | Không | | Viết mới theo D-015 | REF-ONLY (gắn HeroUI/i18n/PostHog/Cloud) | RD-049 | Không | Đã xác minh (mâu thuẫn C-6) |
| Logo, tên, `public/*.png` | OpenHands (Canvas) | **Không** | | | DENY (trademark/brand) | RD-050 | — | Đã xác minh |
| `OpenHands/automation` (cron/webhook, Python + SQLite) | repo khác | Tuỳ chọn, không dùng ở MVP | | Scheduler riêng (BullMQ) | Chưa xác minh | RD-051 | — | Đề xuất |

### 4.4 My Virtual Office (AGPL-3.0-or-later) — chỉ ý tưởng, clean-room

**Quy tắc cứng:** không copy, dịch, "refactor nhẹ" hay dùng nguyên văn bất kỳ file, prompt, CSS, layout JSON, screenshot, favicon nào. Port Python→TypeScript vẫn là derivative work (AGPL §0). Không clone vào monorepo, không symlink, không import. **Không** liệt kê trong THIRD_PARTY_NOTICES (không chứa phần nào của nó); có thể ghi "inspired by" ở README, tự nguyện. Mọi dòng dưới đây có License verdict = `REF-ONLY`/`DENY` và attribution pháp lý = **không áp dụng** (vì không có mã nào được dùng).

| Module / ý tưởng | Repository | Có thể tái sử dụng | Cần chỉnh sửa | Cần viết mới | License verdict | Decision ID | Attribution | Nhãn |
|---|---|---|---|---|---|---|---|---|
| Mã nguồn bất kỳ (`app/server.py`, `app/game.js`, `app/providers/*`…) | my-virtual-office | **Không** | Không | — | DENY | RD-060 / D-012 | Không | Đã xác minh (LICENSE = AGPL v3 đầy đủ) |
| Provider manifest + 22 capability flag | my-virtual-office | Ý tưởng | | `ProviderAdapter.capabilities: Set<Capability>` | REF-ONLY | RD-061 | Không | Đã xác minh (đọc phân tích), Đề xuất |
| CLI adapter pattern (`claude -p stream-json --resume`; `codex app-server --stdio`) | my-virtual-office | Ý tưởng (đều là giao diện CLI công khai) | | `ClaudeCodeAdapter` (Agent SDK) và `CodexAdapter` tự viết | REF-ONLY | RD-062 / D-004, D-005 | Không | Đề xuất; đã được xác nhận độc lập bởi A05/A06 từ tài liệu chính thức |
| Presence coordinator (lifecycle vocabulary, TTL/grace, 6 visual state) | my-virtual-office | Ý tưởng | | Presence service riêng | REF-ONLY | RD-063 | Không | Đề xuất |
| State → vị trí văn phòng (working→desk, meeting→bàn họp, break→bếp, waiting_approval→giơ tay) | my-virtual-office | Ý tưởng | | Mapping riêng trên engine Pixel Agents | REF-ONLY | RD-063 | Không | Đề xuất |
| Workflow Backlog→InProgress→Review→Done, review có cấu trúc, rework cap, escalation | my-virtual-office | Ý tưởng | | `WorkflowService` + PM agent + reviewer độc lập (D-003) | REF-ONLY | RD-064 | Không | Đề xuất |
| Approval bridge (server-request CLI → card UI → respond) | my-virtual-office | Ý tưởng (Codex app-server đã mô tả công khai) | | Approval service | REF-ONLY | RD-064 | Không | Đề xuất |
| A2A comm log hiển thị dạng bubble; meeting bắt buộc summary | my-virtual-office | Ý tưởng | | Message bus + thread | REF-ONLY | RD-065 | Không | Đề xuất |
| Agent Desk file access (resource schema, backup, conflict) | my-virtual-office | Ý tưởng | | File access service path-scoped | REF-ONLY | RD-065 | Không | Đề xuất |
| PromptPack canonical → materialize `CLAUDE.md`/`AGENTS.md` | my-virtual-office | Ý tưởng | | Tự thiết kế | REF-ONLY | RD-066 | Không | Đề xuất |
| Skills library / SKILL.md built-in của MVO | my-virtual-office | **Không** (nội dung AGPL) | | Tự viết skill | DENY (nội dung); chuẩn Agent Skills là mở | RD-060 | Không | Đã xác minh |
| `default-office-config.json`, CSS theme, prompt template nguyên văn | my-virtual-office | **Không** | | Tự vẽ layout/theme | DENY | RD-060 | Không | Đã xác minh |
| A* pathfinding | my-virtual-office | Thuật toán công cộng | | Dùng BFS Pixel Agents (RD-024) hoặc `easystarjs` (MIT) | Không cần tham chiếu MVO | RD-024 | Theo thư viện | Đề xuất |
| Kasm agent browser | my-virtual-office | Image `kasmweb/chromium` (license Kasm riêng) hoặc Playwright | | Script tự viết | REF-ONLY | RD-067 | Theo license Kasm | Đề xuất |
| Chạy MVO như app riêng để demo/so sánh | my-virtual-office | Được (process/container độc lập, không link) | | | ALLOW (AGPL không lan khi chạy cạnh nhau) | RD-068 | Không | Đã xác minh (theo phân tích §10.3) |

### 4.5 Module phải viết mới (không repo nào cung cấp)

| Module | Repository nguồn tham khảo | Có thể tái sử dụng | Cần chỉnh sửa | Cần viết mới | License verdict | Decision ID | Attribution | Nhãn |
|---|---|---|---|---|---|---|---|---|
| Domain company/role/task/assignment/approval + schema PostgreSQL 44 bảng | MVO, ChatDev (ý tưởng) | | | Có (Drizzle, D-002) | Tự sở hữu | RD-070 / D-002, D-016 | Không | Đề xuất |
| Workflow engine "Conductor" (DAG scheduler + BullMQ + semaphore Redis Lua + `WAITING_HUMAN`) | ChatDev (ý tưởng) | | | Có | Tự sở hữu | RD-071 / D-003 | Credit tự nguyện ChatDev | Đề xuất (chưa có prototype) |
| `ClaudeCodeAdapter` trên `@anthropic-ai/claude-agent-sdk` (`canUseTool`, PreToolUse hook) | Pixel Agents (hook/transcript), A05 | Một phần mã hook (RD-018…RD-021) | | Có | Tự sở hữu + MIT phần copy | RD-072 / D-004 | MIT cho phần copy | Đã xác minh (smoke test SDK + OAuth), Đề xuất (thiết kế) |
| `CodexAdapter` (`codex app-server` JSON-RPC, fallback `exec --json`) | MVO (ý tưởng), A06 | | | Có | Tự sở hữu | RD-073 / D-005 | Không | **Đề xuất — chưa chạy thật** (Codex CLI chưa cài) |
| `OpenHandsProvider` (HTTP/WS tới Agent Server) | OpenHands | Client npm | | Có | Tự sở hữu | RD-037, RD-074 / D-006 | Theo RD-039 | Đề xuất |
| Event store append-only + outbox + Socket.IO + Redis Stream | OpenHands (schema), ChatDev (enum) | Tên/shape event | | Có | Tự sở hữu | RD-075 / D-014 | Không | Đề xuất |
| Sandbox Tier 1 (policy + hook + Docker cho build/test) | ChatDev (anti-pattern), OpenHands | | | Có | Tự sở hữu | RD-076 / D-007 | Không | Đề xuất |
| Git worktree isolation + ownership manifest | — | | | Có | Tự sở hữu | RD-077 / D-009 | Không | Đề xuất |
| Scheduler/resource manager (concurrency 5→10→18, host guard, stagger spawn) | — | | | Có | Tự sở hữu | RD-078 / D-010 | Không | Đề xuất; số đo mới tới 3 đồng thời |
| Auth/RBAC, secrets (OS keychain), audit | — | | | Có | Tự sở hữu | RD-079 / D-008 | Không | Đề xuất |
| Phaser `OfficeScene`/`EditorScene` + cầu Zustand/event bus | Pixel Agents (logic) | Logic engine | | Có | Tự sở hữu | RD-080 / D-011 | MIT cho phần port | Đề xuất |
| Asset nhân vật chibi + trạng thái mới (`thinking`, `talking`, `error`, `celebrate`) | Pixel Agents (format) | Format sheet | | Có (vẽ mới, CC0 gốc nếu xác minh) | Tự sở hữu | RD-034 | `CREDITS.json` | Đề xuất |
| Workflow builder React Flow + form sinh từ schema | ChatDev (UX) | | | Có | MIT (`@xyflow/react`) | RD-081 | Giữ badge hoặc mua Pro (quyết định business) | Đề xuất |
| Skill system (`skill.yaml`, nạp `--add-dir`/junction `.agents/skills`) | ChatDev, OpenHands (chuẩn) | Chuẩn mở | | Có | Tự sở hữu | RD-082 / D-013 | Không | Đề xuất |
| Memory (pgvector) + luồng duyệt memory | ChatDev (interface) | Interface | | Có | Tự sở hữu | RD-083 / D-013 | Không | Đề xuất |

---

### 4.6 Thống kê nhanh (đếm theo bảng ở trên)

| Nhóm | Số dòng | Ghi chú |
|---|---|---|
| ChatDev | 20 | Khoảng một nửa là tái sử dụng nội dung/field (RD-001…RD-009), 1 dòng chưa xác minh asset (RD-010), phần còn lại REF-ONLY hoặc không dùng |
| Pixel Agents | 31 | Phần lớn ALLOW (MIT); 3 dòng asset/font chưa an toàn (RD-033, RD-035) |
| OpenHands | 17 | 3 dòng phụ thuộc license chưa xác minh (RD-037…RD-039); UI và brand không dùng |
| My Virtual Office | 14 | 100% REF-ONLY/DENY |
| Viết mới | 15 | Toàn bộ domain, Conductor, adapter, event store, sandbox, Office Phaser |

Thống kê chỉ để định hướng; nguồn sự thật là các bảng ở §4.1–§4.5.

### 4.7 Thứ tự khai thác đề xuất cho Phase 1

| Bước | Việc | RD liên quan | Điều kiện trước |
|---|---|---|---|
| 1 | Tạo `THIRD_PARTY_NOTICES.md`, `licenses/`, header template, CI gate license | RD-001…RD-083 (nền) | Phê duyệt §6 |
| 2 | Định nghĩa `AgentEvent` union + `AgentBrainAdapter` | RD-017, RD-040 | Phê duyệt D-004/D-014 |
| 3 | `ClaudeCodeAdapter` bằng Agent SDK; port hook/transcript cho trường hợp quan sát | RD-018…RD-021, RD-072 | Xác nhận điều khoản subscription |
| 4 | Zod schema workflow tương thích ChatDev + import `ChatDev_v1.yaml` | RD-004, RD-005 | Phê duyệt D-003 |
| 5 | Seed role/phase prompt từ ChatDev 1.0 (đã chỉnh) | RD-001…RD-003 | Có header Apache-2.0 + thay đổi |
| 6 | Port BFS/FSM/layout/seat + manifest asset; Phaser scene | RD-024…RD-031, RD-080 | Chốt asset an toàn (RD-033, RD-034) |
| 7 | `CodexAdapter` | RD-073 | Cài Codex CLI + live test |
| 8 | `OpenHandsProvider` (spike) | RD-037…RD-039, RD-074 | Kiểm license Agent Server |
| 9 | Test bảo mật port từ ChatDev + mock runner từ Pixel Agents | RD-009, RD-032 | Test runner đã chọn (TE-11) |

### 4.8 Ghi chú giải thích một số quyết định khó

- **RD-005 (YAML schema)**: tên field và ngữ nghĩa của một định dạng cấu hình là interface chứ không phải biểu đạt mã; ta giữ tương thích để import mẫu ChatDev, nhưng không dịch từng dòng mã Python validator mà viết lại bằng Zod. Nếu một file TS được dịch sát từ Python của ChatDev, vẫn đính header Apache-2.0.
- **RD-010/RD-033 (sprite)**: cả hai đều nằm trong repo có license cho mã nhưng không có tuyên bố riêng cho art. Quy tắc nhóm: *không có bằng chứng thì không bundle*. Đây là điểm mà các tài liệu nghiên cứu bất đồng (C-2).
- **RD-027 (renderer)**: Pixel Agents khuyến nghị dùng thẳng `renderer.ts` nếu chọn Canvas; D-011 chọn Phaser nên chỉ giữ quy tắc z-sort (`setDepth(zY)`) và logic độc lập renderer. Nếu D-011 bị đảo, nhiều dòng "Chỉ tham khảo" ở §4.2 chuyển sang "Tái sử dụng".
- **RD-037…RD-039 (OpenHands server/client)**: dùng như dịch vụ ngoài giảm nghĩa vụ attribution cho mã, nhưng nếu đóng gói image hoặc phân phối cùng sản phẩm thì nghĩa vụ license áp dụng. Vì chưa đọc license repo `software-agent-sdk`, các dòng này ghi **chưa xác minh**.
- **RD-062 (CLI adapter pattern của MVO)**: lệnh `claude -p --output-format stream-json --resume` và `codex app-server --stdio` là giao diện công khai của chính hai CLI; ta dựa vào tài liệu chính thức và smoke test của A05/A06, không dựa vào mã MVO. Điều này giúp bảo vệ clean-room.
- **RD-068 (chạy MVO riêng)**: chỉ dùng để so sánh hành vi; không đưa vào repo, không gọi API của nó từ sản phẩm.

## 5. Hướng dẫn thực thi attribution

1. **Header mỗi file copy/port** (theo `LICENSE_AND_REUSE.md` §3.2):
   ```ts
   // Portions derived from <repo> (<URL>), Copyright (c) <year> <holder>, licensed under <MIT|Apache-2.0>.
   // Modified by AI WORKSPACE team: <tóm tắt thay đổi>.   // bắt buộc với Apache-2.0
   ```
2. **Phạm vi bắt buộc ghi thay đổi** (Apache-2.0 §4(b)): mọi file lấy từ ChatDev (kể cả prompt JSON/YAML) phải có ghi chú đã sửa.
3. **Ghi rõ nguồn commit/tag** trong header: ChatDev `a2ea4a2` (working tree) hoặc `v1.1.6` (prompt 1.0); Pixel Agents v1.5.0 (từ `add749a` trở về sau); Agent Canvas 1.34.0 (`7219c19`).
4. **Màn hình "About → Open-source licenses"** trong app render `THIRD_PARTY_NOTICES.md`.
5. **Không** đưa MVO vào NOTICES.

## 6. Kế hoạch THIRD_PARTY_NOTICES

| Hạng mục | Nội dung | Thời điểm | Người chịu trách nhiệm đề xuất |
|---|---|---|---|
| File `THIRD_PARTY_NOTICES.md` ở root | Dùng template ở `LICENSE_AND_REUSE.md` §4; 3 mục dự án: ChatDev (Apache-2.0, `Copyright 2025 OpenBMB`), Pixel Agents (MIT, `Copyright (c) 2026 Pablo De Lucca`), OpenHands Agent Canvas (MIT, `Copyright © 2025 OpenHands contributors`) | Tạo ở Phase 1, trước commit đầu tiên chứa nội dung bên thứ ba | Tech lead |
| Thư mục `licenses/` | `Apache-2.0.txt`, `MIT.txt`, `OFL.txt`, `CC0.txt` | Cùng thời điểm | Tech lead |
| Mục asset | Sinh tự động từ `CREDITS.json` cạnh mỗi thư mục asset (`author`, `source`, `license` SPDX, `modified`) | Khi asset đầu tiên được thêm | Asset owner |
| Mục dependency | `THIRD_PARTY_NOTICES.deps.txt` sinh bằng `pnpm licenses list --json` (và `cargo about` khi có Tauri) | Mỗi build/release | CI |
| Mục font | Press Start 2P (OFL 1.1), IBM Plex Mono (OFL 1.1); font design hiện chọn Inter + JetBrains Mono (D-015) cũng cần thêm vào | Khi bundle font | Designer |
| Mục JIK-A-4 MetroCity | Chỉ thêm khi đã chụp bằng chứng CC0; ghi tên tác giả dù không bắt buộc | Sau khi xác minh C-2 | Asset owner |
| Mục Agent Server / client OpenHands | Thêm khi RD-037/RD-039 được xác minh và được phân phối cùng sản phẩm | Trước khi phát hành D-006 | Tech lead |
| CI gate | Chặn dependency `AGPL*`, `GPL*`, `SSPL`, `BUSL`, `UNKNOWN`; chặn asset thiếu `CREDITS.json` hoặc license ngoài allowlist (OWR-2 Q-15: mặc định CC0-1.0; OFL-1.1 chỉ font kèm NOTICE; CC-BY-4.0 chỉ package Owner đã duyệt; MIT asset không rõ nguồn không nằm trong allowlist mặc định) | Khi có CI (`docs/development/CI_CD.md`) | DevOps |
| Kiểm tra apache NOTICE | Nếu ChatDev thêm file `NOTICE` về sau, chép vào mục ChatDev (hiện không có) | Mỗi lần cập nhật tham chiếu | Tech lead |

Trạng thái: **Đề xuất**. Chưa có file `THIRD_PARTY_NOTICES.md` hay `licenses/` nào được tạo trong Phase 0 (Phase 0 chỉ có tài liệu).

---

## 7. Rủi ro

| ID | Rủi ro | Mức | Giảm thiểu |
|---|---|---|---|
| RR-1 | Developer đọc mã MVO rồi viết lại → derivative AGPL, buộc mở nguồn toàn bộ sản phẩm | Cao | Quy trình clean-room; code review có checklist; cấm đường dẫn `my-virtual-office` trong repo |
| RR-2 | Bundle asset không rõ license (JIK-A-4, 12 sprite ChatDev, furniture Pixel Agents, font FS Pixel Sans) | Trung bình–Cao | Chỉ asset có `CREDITS.json`; thay font; xin xác nhận tác giả pixel-agents |
| RR-3 | Lấy nhầm asset từ git history cũ của pixel-agents (tileset proprietary trước `add749a`) | Cao | Cấm checkout PNG từ lịch sử; chỉ copy từ HEAD tại thời điểm đã kiểm tra |
| RR-4 | Thiếu ghi chú thay đổi cho file Apache-2.0 | Thấp | Header bắt buộc + CI lint header |
| RR-5 | Agent Server/typescript-client có license khác MIT hoặc có phần enterprise | Trung bình | Kiểm tra license trước khi phụ thuộc; ghi vào NOTICES |
| RR-6 | Wire contract Agent Server đổi nhanh → provider hỏng | Trung bình | Version-gate, feature probe, fallback 404 |
| RR-7 | Mã copy từ Pixel Agents mang theo giả định Claude-specific và global state | Trung bình | Tách interface (RD-017), viết test với mock runner (RD-032) |
| RR-8 | Nhầm "Đề xuất" với "đã duyệt" khi bắt đầu Phase 1 | Trung bình | Toàn bộ RD-xxx ở trạng thái Chờ duyệt cho tới khi có chữ ký chủ sở hữu |

## 8. Vấn đề chưa giải quyết

1. License và nguồn gốc 12 sprite ChatDev (RD-010).
2. Bằng chứng CC0 của JIK-A-4 (RD-033, mâu thuẫn C-2) — cần ảnh chụp trang itch.io tại ngày kiểm tra.
3. License của `software-agent-sdk`, `typescript-client`, image `ghcr.io/openhands/agent-server` (RD-037…RD-039).
4. Xác nhận bằng văn bản của tác giả pixel-agents về furniture/floor/wall/pet "original" (hành động đề xuất ở `LICENSE_AND_REUSE.md` §6.3).
5. Quyết định business về badge React Flow (RD-081).
6. Có cần luật sư rà soát verdict AGPL/Apache trước khi phát hành thương mại không.
7. Có cần dùng `OpenHands/automation` hay tự làm scheduler (RD-051) — đề xuất tự làm.

## 9. Cần người dùng phê duyệt

- [ ] Phê duyệt toàn bộ bảng RD-001…RD-083 (hiện tất cả là Đề xuất).
- [ ] Xác nhận "MVO = REF-ONLY + clean-room" và cho phép đưa quy tắc này vào ADR.
- [ ] Phê duyệt việc **không** bundle: sprite ChatDev (RD-010), nhân vật JIK-A-4 cho tới khi có bằng chứng (RD-033), font FS Pixel Sans (RD-035).
- [ ] Cho phép kiểm tra license Agent Server/typescript-client trước khi chấp nhận D-006.
- [ ] Phê duyệt kế hoạch THIRD_PARTY_NOTICES (§6) và CI gate license.
- [ ] Quyết định có hỏi tác giả pixel-agents để xác nhận nguồn gốc art hay không.
