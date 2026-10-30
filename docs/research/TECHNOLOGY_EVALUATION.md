# TECHNOLOGY_EVALUATION — Hồ sơ quyết định công nghệ (Decision Records)

> Tác giả: Technical Research Lead (tổng hợp Phase 0). Ngày: 2026-10-30.
> Nguồn chính: `docs/DATABASE_DESIGN.md` §1, `docs/research/{ORCHESTRATION_WORKFLOW_RESEARCH, CLAUDE_CODE_INTEGRATION, CODEX_INTEGRATION, LICENSE_AND_REUSE, MULTI_AGENT_SCALABILITY, PIXEL_AGENTS_ANALYSIS, OPENHANDS_ANALYSIS}.md`, `docs/development/CI_CD.md`, `docs/design/*`, `DECISION_LOG.md` (thư mục gốc repo).
> Nhãn bằng chứng: **[Đã xác minh]** = đã chạy lệnh/đọc mã/đọc file cấu hình thật trong Phase 0; **[Tài liệu]** = dựa trên tài liệu chính thức hoặc kết quả web search do agent khác ghi lại, chưa tự kiểm chứng; **[Chưa có bằng chứng]** = không có nguồn nào trong Phase 0.
> Trạng thái quyết định: **Đề xuất** (nhóm nghiên cứu khuyến nghị) hoặc **Cần phê duyệt** (có ràng buộc pháp lý/chi phí/rủi ro cao, hoặc đang xung đột giữa các tài liệu). Không quyết định nào đã được chủ sở hữu phê duyệt.

---

## 1. Mục đích

Ghi lại, cho từng lựa chọn công nghệ lớn của AI WORKSPACE: các phương án đã xét, tiêu chí, bằng chứng (và mức độ tin cậy của bằng chứng), khuyến nghị, trạng thái — để chủ sở hữu có thể phê duyệt hoặc bác bỏ có cơ sở, và để Phase 1 không phải tranh luận lại.

## 2. Phạm vi

13 quyết định: ORM, workflow engine, tích hợp Claude, tích hợp Codex, game engine, desktop shell, state/query, realtime transport, monorepo tool, package manager, test framework (Vitest, Playwright), charting library, Monaco. Không bao gồm: chọn database (đã có D-002), thiết kế UI (docs/design), hạ tầng triển khai.

## 3. Bảng tổng hợp và cờ thiếu bằng chứng

| # | Quyết định | Khuyến nghị | ID liên quan | Trạng thái | Độ tin cậy bằng chứng | Cờ |
|---|---|---|---|---|---|---|
| TE-01 | ORM | Drizzle (postgres.js) | D-002 | APPROVED (Owner, 2026-10-30) | Trung bình: so sánh trên tài liệu, chưa có prototype migration | Xung đột C-1: tài liệu orchestration dùng Prisma |
| TE-02 | Workflow engine | "Conductor" tự xây trên BullMQ + Postgres | D-003 | Cần phê duyệt | Trung bình: phân tích thiết kế, chưa có prototype | Chi phí tự xây lớn; Inngest license "cần verify" |
| TE-03 | Tích hợp Claude | `@anthropic-ai/claude-agent-sdk` | D-004 | Đề xuất | **Cao** cho kỹ thuật (smoke test thật) | Điều khoản dùng OAuth subscription cho tự động hoá: **chưa xác minh** |
| TE-04 | Tích hợp Codex | `codex app-server`; fallback `exec --json` | D-005 | Cần phê duyệt | **Thấp**: CLI chưa cài, chưa chạy live test nào | **Thiếu bằng chứng chạy thật** |
| TE-05 | Game engine | Phaser 3.90 → 4.x | D-011 | Đề xuất | Thấp–Trung bình: số liệu phiên bản từ web search; chưa prototype | Mâu thuẫn C-4 (Pixel Agents dùng Canvas) |
| TE-06 | Desktop shell | MVP web + NestJS local; Tauri 2 ở phase sau | D-001 | Cần phê duyệt | Trung bình: Electron của OpenHands đã đọc mã; Tauri chỉ từ tài liệu | Sidecar Node/native module chưa thử |
| TE-07 | State/query | Zustand 5 + TanStack Query 5 | — | Đề xuất | Trung bình–Cao: 2 repo tham chiếu dùng (OpenHands) | — |
| TE-08 | Realtime | Socket.IO 4 (NestJS gateway) | D-014 | Đề xuất | Trung bình: so sánh lý thuyết; hai repo tham chiếu dùng `ws` | Chưa kiểm tra Socket.IO + Redis adapter |
| TE-09 | Monorepo tool | pnpm workspaces + Turborepo | D-001, CI_CD.md | Cần phê duyệt | **Thấp**: Turborepo chỉ xuất hiện trong `CI_CD.md`, không có nghiên cứu | **Thiếu bằng chứng**; D-001 không nhắc Turborepo |
| TE-10 | Package manager | pnpm 10 | D-001 | Đề xuất | Trung bình: pnpm 10.34.6 đã cài trên máy dev | Native module (node-pty, sqlite) trên Windows chưa thử |
| TE-11 | Test framework | Vitest (unit/integration), Playwright (E2E) | — | Đề xuất | Trung bình: cả hai được dùng trong 2 repo tham chiếu | Vitest + NestJS decorator metadata cần spike; Playwright + Tauri/WebView2 chưa thử |
| TE-12 | Charting | Recharts (qua shadcn/ui chart) — tạm thời | — | Cần phê duyệt | **Không có**: không tài liệu nào đánh giá, không repo tham chiếu dùng | **Thiếu bằng chứng hoàn toàn** |
| TE-13 | Monaco | `@monaco-editor/react` | — | Đề xuất | Trung bình: OpenHands dùng `@monaco-editor/react 4.7.0` + `monaco-editor 0.56.0` | Cấu hình loader offline/Tauri CSP, kích thước bundle chưa kiểm |

Tóm tắt cờ: **thiếu bằng chứng nghiêm trọng** ở TE-04, TE-09, TE-12; **xung đột tài liệu** ở TE-01, TE-05, TE-09; **rủi ro điều khoản** ở TE-03.

---

## 4. Decision records

### TE-01 — ORM

| Phương án | Ưu | Nhược |
|---|---|---|
| **Drizzle ORM** + `drizzle-kit` + postgres.js | SQL thuần khi generate migration; partial index, `jsonb().$type<T>()`, transaction lồng bằng savepoint, không binary engine | Relational query API kém chín hơn; không có package NestJS chính thức (cần tự viết `DrizzleModule` ~30 dòng) |
| Prisma 6/7 | Hệ sinh thái chín; `nestjs-prisma`; Prisma Studio | Không mô tả được partial index, bảng partition, trigger, CHECK trong schema; migrate diff có thể xung đột; transaction timeout mặc định 5 s |
| TypeORM | Quen thuộc với NestJS | Không được đánh giá trong tài liệu nào; `ENVIRONMENT_SETUP.md` chỉ liệt kê như lựa chọn mở |

**Tiêu chí**: hỗ trợ event store phân vùng theo tháng + partial unique index cho soft delete; transaction `appendEvent + update aggregate + outbox`; chạy trên Windows với đường dẫn có khoảng trắng (`Agent management`); type safety; migration chỉnh sửa được.

**Bằng chứng**: `DATABASE_DESIGN.md` §1.1 (12 tiêu chí) và §1.2 (5 lý do) **[Tài liệu]**. Chưa có prototype migration với partition; chưa đo hiệu năng. Các phát biểu về phiên bản (Drizzle v1.x, Prisma 7 bỏ Rust engine) đến từ nghiên cứu web của agent khác, tôi chưa tự kiểm chứng.

**Xung đột**: `ORCHESTRATION_WORKFLOW_RESEARCH.md` đề bài là "PostgreSQL (Prisma)", §3.2 có schema Prisma và code `prisma.$transaction`; `AGENT_ORGANIZATION.md` ghi quyền sở hữu đường dẫn `prisma/**`; `ENVIRONMENT_SETUP.md` để mở ("Prisma/Drizzle/TypeORM"). `DECISION_LOG.md` D-002 chọn Drizzle.

**Khuyến nghị**: Drizzle. Dịch schema Prisma trong tài liệu orchestration sang Drizzle khi triển khai; sửa đường dẫn `prisma/**` thành `packages/db/**` (hoặc tên được duyệt) trong `AGENT_ORGANIZATION.md`.

**Trạng thái**: **Cần phê duyệt** (vì xung đột tài liệu và chi phí đổi ORM sau này cao).

**Spike đề xuất** trước khi khoá: 1 ngày, tạo bảng `agent_events` partition theo tháng + partial unique index bằng `drizzle-kit generate --custom`, chạy migrate hai lần để chắc không drift.

---

### TE-02 — Workflow engine

| Phương án | Ưu | Nhược |
|---|---|---|
| **Conductor** (DAG executor tự xây trên BullMQ + Postgres) | JSON là định nghĩa (khớp React Flow); durable state trong Postgres của ta; approval `WAITING_HUMAN` không giữ worker; semaphore Redis Lua xuyên replica; ops thấp (đã có Redis + PG) | Phải tự xây scheduler, recovery, test chaos; rủi ro bug về idempotency |
| LangGraph JS + PostgresSaver | Có sẵn interrupt/resume, conditional edges | `interrupt()` chạy lại node từ đầu (subprocess 10–40 phút); `Send` không cap xuyên process; graph là code; checkpoint dạng blob khó báo cáo |
| Temporal TS | Replay deterministic, mạnh nhất về độ bền | Server 4 service + DB (+ Elasticsearch tùy chọn); workflow là code, không hợp JSON user-editable; overkill cho ~10 agent |
| XState v5 + BullMQ | Typed state machine | DAG ↔ statechart không tự nhiên; tự lưu snapshot |
| Trigger.dev / Inngest | Durable function runner | Chỉ thay lớp BullMQ, không thay DAG interpreter; Inngest self-host: nguồn mâu thuẫn về license |
| BullMQ Flows thuần | Có sẵn | Là tree, không có diamond/condition edge/wait-human |

**Tiêu chí**: R1–R10 của `ORCHESTRATION_WORKFLOW_RESEARCH.md` §1 (durable PG, resume sau restart, approval chờ ngày, fan-out có cap, retry, cancel/pause, branching, graph từ React Flow JSON, versioned definition, event cho UI) và chi phí vận hành.

**Bằng chứng**: ma trận §1 **[Tài liệu]** kèm trích dẫn docs LangGraph/BullMQ/Temporal; đối chiếu với mã ChatDev (`workflow/graph.py`, `topology_builder.py`) **[Đã xác minh bởi agent 01/07]**. Chưa có prototype scheduler. Việc XState nên chỉ dùng cho lifecycle từng node/run là đề xuất.

**Khuyến nghị**: Conductor; XState v5 chỉ để validate transition của `node_run`/`workflow_run`. Giữ LangGraph như tuỳ chọn *bên trong* một node (không làm engine).

**Rủi ro riêng**: Windows không có SIGTERM tin cậy cho cây process CLI (dùng `taskkill /T /F`/`tree-kill`); semaphore leak khi worker chết (lease TTL + reaper); 2 scheduler tick cùng run (`SELECT FOR UPDATE` + `jobId` dedupe).

**Trạng thái**: **Cần phê duyệt** — đây là phần lớn công sức Phase 1; nên yêu cầu spike "scheduler + tool node giả" (bước 2 trong thứ tự build §10) trước khi cam kết.

---

### TE-03 — Tích hợp Claude Code

| Phương án | Ưu | Nhược |
|---|---|---|
| **`@anthropic-ai/claude-agent-sdk` `query()`** (0.3.314) | Kế thừa OAuth login (đã thấy `apiKeySource: "none"`); `canUseTool` in-process; hook typed; `interrupt()`, `streamInput()`; bundle binary cùng version (deterministic) | API lớn, đổi nhanh (0.3.x); `env` thay toàn bộ env; binary lớn theo platform |
| Raw `spawn('claude', ['-p', '--output-format', 'stream-json'])` | Zero dependency; ít trừu tượng | Phải tự dựng MCP `--permission-prompt-tool` hoặc HTTP hook cho approval; tự parse NDJSON; phụ thuộc phiên bản `claude` trên PATH; hủy chỉ bằng SIGTERM |
| Hook + transcript tail (kiểu Pixel Agents) | Quan sát session do người dùng tự chạy | Chỉ quan sát, không điều khiển; format JSONL đổi theo version |
| Agent SDK qua OpenHands ACP | Chạy trong sandbox | Thêm tầng Python/uv; chưa thử |

**Tiêu chí**: dùng được đăng nhập subscription sẵn có; approval gating; streaming; cancel; resume; ổn định phiên bản; chạy ổn trên Windows (không dùng argv dài).

**Bằng chứng** **[Đã xác minh]** (agent 05, ngày 2026-10-30): CLI 2.1.312, SDK 0.3.314; `claude -p --output-format json`/`stream-json --verbose` chạy OK; `--resume` giữ `session_id`; `--input-format stream-json` multi-turn OK; `--json-schema` có `structured_output`; `canUseTool` chặn được `Write` (`permission_denials = 1`); 5 process song song cùng login 7 s không lỗi; SDK chạy với `ANTHROPIC_API_KEY` không đặt.

**Cờ**: các smoke test dùng prompt nhẹ. Chưa đo task dài, chưa kiểm tra hành vi khi chạm quota 5 giờ/tuần. **Chưa có bằng chứng** về việc điều khoản dịch vụ của gói subscription cho phép dùng đăng nhập OAuth qua SDK trong một sản phẩm đa-agent — `DECISION_LOG.md` mục "Mâu thuẫn và khoảng trống" #2 ghi nhận đúng điểm này.

**Khuyến nghị**: Agent SDK làm adapter chính; giữ `RawCliClaudeAdapter` nhỏ cho `claude auth status` và `--version`. Truyền prompt dài qua stdin/stream-json, không qua argv (Windows `ENAMETOOLONG`). Ghim version SDK, kiểm contract bằng fixture.

**Trạng thái**: **Đề xuất** (kỹ thuật đã xác minh), kèm điều kiện: chủ sở hữu xác nhận điều khoản/quota trước khi triển khai đa session.

---

### TE-04 — Tích hợp Codex

| Phương án | Ưu | Nhược |
|---|---|---|
| **`codex app-server`** (JSON-RPC stdio) | Nhiều thread trên một process; approval tương tác; delta token; interrupt/steer; đọc rate limit/account/model | Đánh dấu **[experimental]**; binding sinh theo version; phức tạp hơn |
| `codex exec --json` | API ổn định hơn; đơn giản | 1 process = 1 turn; không approval tương tác; không delta; không interrupt |
| `@openai/codex-sdk` (0.166.0) | Typed, `runStreamed` | Chỉ wrapper `codex exec`; không approval callback; mỗi `run()` một process; coi như fallback/batch |

**Tiêu chí**: reuse ChatGPT login; approval gating; nhiều thread song song; đọc quota; chạy native Windows.

**Bằng chứng**: **Thiếu bằng chứng chạy thật.** `CODEX_INTEGRATION.md` §0: `codex` không có trên PATH; `@openai/codex` chưa cài global; chỉ có binary bundled của extension VS Code (`codex-cli 0.164.0-alpha.11.1`) dùng để đọc `--help`, `login status` ("Logged in using ChatGPT") và `app-server generate-ts/generate-json-schema`. Không có session nào được tạo. Các lợi thế của `app-server` suy ra từ schema sinh ra và tài liệu **[Tài liệu]**, không từ chạy thử.

**Khuyến nghị**: `app-server` làm đường chính, `exec --json` làm dự phòng, SDK chỉ cho batch. Concurrency mặc định 2–3 turn/account; throttle khi `usedPercent > 80`.

**Rủi ro**: API experimental đổi theo version; ChatGPT plan là hạn mức dùng chung (cửa sổ 5 giờ + tuần, **hard ceiling**); ToS fair-use khi chạy "công ty AI" trên một tài khoản cá nhân (chưa xác minh).

**Trạng thái**: **Cần phê duyệt** và **bắt buộc live test** (`CODEX_INTEGRATION.md` §11.4) sau khi người dùng cài `npm i -g @openai/codex` — trước đó mọi chi tiết adapter là giả thuyết.

---

### TE-05 — Game engine cho Virtual Office

| Phương án | Ưu | Nhược |
|---|---|---|
| **Phaser 3.90 (MVP) → 4.x** | Tilemap (Tiled), camera, input, animation có sẵn; ví dụ nhiều; template React chính thức | Bundle ~1.2 MB gzip (v3); v4 còn mới; giao tiếp với React qua event bus/store |
| PixiJS 8 + `pixi-viewport` + `easystarjs` | Nhẹ hơn (~150–300 KB); `@pixi/react` v8 | Phải ghép tilemap/viewport từ plugin cộng đồng |
| Canvas 2D thuần | 0 phụ thuộc; **Pixel Agents đã chạy ổn 10–20 nhân vật**; tái dùng `renderer.ts` | Tự viết camera, input, tilemap, editor; dễ tái phát minh |

**Tiêu chí**: 10–50 nhân vật + tilemap tối đa ~100×100; zoom/pan; layout editor kéo-thả; pixel-art; tích hợp React; khả năng tái dùng mã Pixel Agents.

**Bằng chứng**: `LICENSE_AND_REUSE.md` §5.1 **[Tài liệu]** (web search: Phaser 3.90.0 05/2025; Phaser 4.0.0 "Caladan" 10/04/2026; PixiJS 8.19.0 07/2026). Tôi chưa kiểm chứng các số phiên bản. **[Đã xác minh]** riêng điểm Pixel Agents dùng Canvas thuần (`PIXEL_AGENTS_ANALYSIS.md` §0, §4.1). Chưa có prototype Phaser.

**Mâu thuẫn C-4**: `PIXEL_AGENTS_ANALYSIS.md` §8.1 khuyến nghị dùng thẳng `renderer.ts` nếu Canvas; D-011 chọn Phaser. Hệ quả: chỉ port logic (BFS, FSM, layout, seat), viết lại renderer.

**Khuyến nghị**: Phaser 3.90 cho MVP; cô lập Phaser sau interface `OfficeRenderer` để có thể đổi sang Pixi/Canvas nếu bundle hoặc hiệu năng không đạt. Không tạo `Phaser.Game` trong render body; tạo trong `useEffect`, hủy ở cleanup (React 19 StrictMode mount hai lần ở dev).

**Trạng thái**: **Đề xuất**. Spike 2–3 ngày: tilemap 40×30 + 30 sprite + BFS + zoom/pan để xác nhận khung giờ 60 fps.

---

### TE-06 — Desktop shell

| Phương án | Ưu | Nhược |
|---|---|---|
| **Phase 1: web app + NestJS chạy local; Phase 2: Tauri 2** | Không bị chặn bởi packaging/sidecar/code-signing; CLI được spawn từ NestJS (Node) nên shell không cần quyền spawn; installer nhỏ; capability model tốt | Rust cho phần shell; NestJS phải đóng gói sidecar (Node SEA/`@yao-pkg/pkg`/ship Node runtime); WebView2 chỉ chắc trên Windows |
| Electron | Node sẵn có (`fork()` NestJS); Chromium đồng nhất; OpenHands có mẫu `electron/main.mjs` | 80–240 MB installer, ~200–300 MB RAM; cấu hình bảo mật thủ công; OpenHands vẫn phải bundle thêm Node + uv |
| Chỉ web (không có desktop) | Đơn giản nhất | Không có trải nghiệm ứng dụng, khó quản lý process nền |

**Bằng chứng**: Electron của OpenHands **[Đã xác minh]** (đọc `electron/main.mjs`, `electron-builder.config.mjs`, `download-uv.mjs`, `download-node.mjs`; bài học: bundle Node thật, PATH tối thiểu khi mở từ GUI, kill process tree, chờ `/server_info`). Số liệu Tauri (installer 8–15 MB, RAM 30–50 MB, sidecar Node) **[Tài liệu]**.

**Khuyến nghị**: theo D-001. Tauri chỉ ở phase sau; trước đó cần spike "NestJS sidecar + `node-pty`/native module ABI trên Windows".

**Trạng thái**: **Cần phê duyệt** (ảnh hưởng kiến trúc đóng gói và kỹ năng đội: Rust).

---

### TE-07 — State và query libraries

| Phương án | Ưu | Nhược |
|---|---|---|
| **Zustand 5 + TanStack Query 5** | ~0,5 KB; dùng được ngoài React (Phaser scene đọc `getState()`/`subscribe`); cache invalidation theo WS event | Hai thư viện thay vì một |
| Redux Toolkit 2 + RTK Query | Devtools, time-travel | ~13 KB; boilerplate; Phaser khó đọc store |
| Chỉ React Context / SWR | Ít phụ thuộc | Không hợp với đồng bộ Phaser và cache phức tạp |

**Bằng chứng**: `LICENSE_AND_REUSE.md` §5.4 **[Tài liệu]**; **[Đã xác minh]** `OpenHands/package.json` dùng `zustand 5.0.14` và `@tanstack/react-query 5.103.1` (và plugin ESLint của TanStack Query) — nghĩa là cặp này chạy trong một sản phẩm cùng loại.

**Khuyến nghị**: Zustand (client state: selection, UI, editor mode, cache runtime của agent) + TanStack Query (server state). Undo/redo layout dùng logic thuần của editor Pixel Agents (RD-029), không cần Redux.

**Trạng thái**: **Đề xuất**.

---

### TE-08 — Realtime transport

| Phương án | Ưu | Nhược |
|---|---|---|
| **Socket.IO 4** (`@nestjs/platform-socket.io`) | Rooms/namespaces theo `project/agent/run`; auto-reconnect; ack cho lệnh điều khiển | Client không nói chuyện với native WebSocket; scale nhiều instance cần Redis adapter + sticky session; ~10 KB client |
| `ws` thuần (`@nestjs/platform-ws`) | Nhẹ; client native; **Pixel Agents (`ws ^8.18.0`) và OpenHands (`ws 8.21.0`) đều dùng** | Tự viết reconnect/rooms/ack |
| SSE | Đơn giản, một chiều | Không có kênh điều khiển ngược |

**Tiêu chí**: reconnect đáng tin khi backend restart; replay theo `lastEventId`/`after_seq`; ack lệnh điều khiển; tách payload JSON `{type, runId, ts, data}` khỏi transport để đổi sau.

**Bằng chứng**: `LICENSE_AND_REUSE.md` §5.4 **[Tài liệu]** (lập luận). D-014 chọn outbox → Redis pub/sub (Socket.IO) + Redis Stream cho Office. Pixel Agents có `MessageTransport` + `webSocketTransport.ts` (backoff + queue) **[Đã xác minh]** — có thể dùng làm mẫu nếu đổi sang `ws`. Chưa có prototype hay đo tải.

**Lưu ý**: kết nối backend→OpenHands Agent Server là WS thuần (`/sockets/session/{id}?after_seq=N`), độc lập với lựa chọn cho browser↔NestJS.

**Khuyến nghị**: Socket.IO cho MVP cục bộ single-instance; giữ lớp `EventTransport` trừu tượng.

**Trạng thái**: **Đề xuất**.

---

### TE-09 — Monorepo tool

| Phương án | Ưu | Nhược |
|---|---|---|
| **pnpm workspaces thuần** (D-001) | Ít phụ thuộc, đơn giản | Không có cache task, không có task graph |
| **pnpm workspaces + Turborepo** (`CI_CD.md`) | Cache `.turbo/`, `turbo lint typecheck` có filter theo thay đổi | Thêm một công cụ; cần cấu hình pipeline |
| Nx | Mạnh, generators | Nặng hơn, đường cong học |

**Bằng chứng**: **Gần như không có.** Turborepo chỉ xuất hiện trong `docs/development/CI_CD.md` (§ đầu: "monorepo pnpm + Turborepo", "[Cần người dùng phê duyệt]") và cache `.turbo/`; không tài liệu nghiên cứu nào so sánh Nx/Turborepo/thuần pnpm; 4 repo tham chiếu không dùng (`OpenHands` dùng npm đơn gói). **D-001 chỉ ghi "pnpm workspaces"**, không nhắc Turborepo.

**Khuyến nghị tạm thời**: bắt đầu với pnpm workspaces; thêm Turborepo khi số package > ~5 hoặc thời gian CI > ngưỡng đặt ra — vì thêm sau rẻ, bỏ đi sau cũng rẻ.

**Trạng thái**: **Cần phê duyệt**; **cờ thiếu bằng chứng**. Cần người phụ trách CI/CD xác nhận hoặc cung cấp so sánh.

---

### TE-10 — Package manager

| Phương án | Ưu | Nhược |
|---|---|---|
| **pnpm 10** | Workspace tốt; `pnpm licenses list --json` phục vụ THIRD_PARTY_NOTICES; store tiết kiệm đĩa; `packageManager` + corepack | Quy tắc hoisting nghiêm có thể làm vài gói lỗi; native module cần `onlyBuiltDependencies` |
| npm 11 | OpenHands dùng (`npm@11.12.1`) | Workspace kém hơn; không có `licenses` tương đương |
| Yarn / Bun | — | Không được đánh giá trong tài liệu nào |

**Bằng chứng**: pnpm `10.34.6` đã cài global trên máy dev **[Đã xác minh]** (`CODEX_INTEGRATION.md` §0); Node v24.19.0 **[Đã xác minh]** (`MULTI_AGENT_SCALABILITY.md` §4.1). Chưa thử cài native module (`node-pty`, `better-sqlite3`) trên Windows với đường dẫn có khoảng trắng.

**Khuyến nghị**: pnpm 10 + `.nvmrc` Node 24 + `packageManager` field.

**Trạng thái**: **Đề xuất**.

---

### TE-11 — Test frameworks (Vitest, Playwright)

| Phương án | Ưu | Nhược |
|---|---|---|
| **Vitest** | Dùng bởi Pixel Agents (`vitest ^4.1.10`, cả server và webview) và OpenHands (`vitest 5.0.0` + `@vitest/coverage-v8`); nhanh; tương thích Vite | NestJS mặc định Jest; decorator metadata cần plugin SWC — cần spike |
| Jest | Mặc định NestJS; `CHATDEV_ANALYSIS.md` §11.1 đề xuất port test bảo mật sang Jest | Cấu hình ESM/TS nặng hơn; hai runner nếu frontend dùng Vitest |
| **Playwright** | Dùng bởi Pixel Agents (`@playwright/test ^1.58.2`) và OpenHands (1.63.0, nhiều config mock-llm, bind-policy, docker) | E2E cho Tauri/WebView2 không phải lúc nào cũng chạy được bằng Playwright thông thường |
| Cypress | — | Không được đánh giá |

**Bằng chứng** **[Đã xác minh]** từ `package.json` của hai repo (đã đọc trực tiếp). OpenHands còn dùng Stryker (mutation) và Testing Library. Pixel Agents có mock Claude runner cho E2E (RD-032) — mẫu rất hợp để test adapter mà không tốn quota.

**Khuyến nghị**: Vitest cho `apps/web` và `packages/*`; cho `apps/api` bắt đầu bằng Vitest + `unplugin-swc` nếu spike qua, nếu không thì Jest cho api. Playwright cho E2E web; mock Claude/Codex runner dựa trên mẫu Pixel Agents. Ghi vào ADR để chốt một runner duy nhất cho mỗi package.

**Cờ**: kết luận về Vitest + NestJS decorators và Playwright + Tauri là **[Chưa có bằng chứng]** từ Phase 0 — cần spike.

**Trạng thái**: **Đề xuất**.

---

### TE-12 — Charting library

| Phương án | Ưu | Nhược |
|---|---|---|
| **Recharts** (qua `shadcn/ui` chart) | Gần stack đã chọn (shadcn/ui, Tailwind v4, React 19); API khai báo; đủ cho line/bar/area | Hiệu năng giảm với nhiều điểm dữ liệu; tuỳ biến sâu khó |
| visx | Linh hoạt, nhẹ | Phải tự ghép nhiều; tốn công |
| ECharts | Mạnh, nhiều chart/performance tốt | Lớn; ít "React-native" |
| Chart.js / Tremor | — | Chưa được xem xét |

**Bằng chứng**: **Không có.** Không tài liệu nào trong `docs/research` đánh giá chart; `grep` trong `package.json` của Pixel Agents và OpenHands không thấy thư viện chart. Nhu cầu đã biết: dashboard cost/token/usage (`DATABASE_DESIGN.md` §6 usage aggregation) và gauge quota. Đề xuất Recharts chỉ dựa trên khả năng tương thích với shadcn/ui, **chưa kiểm chứng** phiên bản hay hiệu năng.

**Khuyến nghị**: tạm chọn Recharts cho MVP; bao bọc sau component nội bộ `<MetricChart>` để đổi được. Nếu dashboard theo chuẩn trực quan hoá riêng của tổ chức (ví dụ chuẩn IBCS), cần đối chiếu trước khi chốt.

**Trạng thái**: **Cần phê duyệt**; **cờ: thiếu bằng chứng hoàn toàn**. Cần một spike 1 ngày so Recharts với visx/ECharts trên dữ liệu usage giả (≥ 5k điểm).

---

### TE-13 — Monaco editor

| Phương án | Ưu | Nhược |
|---|---|---|
| **`@monaco-editor/react` + `monaco-editor`** | Quen thuộc (lõi VS Code); diff editor sẵn; yêu cầu thiết kế dùng Monaco cho system prompt (versioned, diff), file editor, diff viewer | Nặng (hàng MB); mặc định loader tải từ CDN — cần cấu hình bundle cục bộ cho offline/CSP Tauri; không hỗ trợ touch nhỏ |
| CodeMirror 6 | Nhẹ, modular | Không có diff editor tích hợp tương đương; chưa đánh giá |
| `<textarea>` + highlight | Rất nhẹ | Thiếu tính năng |

**Bằng chứng**: **[Đã xác minh]** `OpenHands/package.json`: `@monaco-editor/react 4.7.0`, `monaco-editor 0.56.0`. Yêu cầu thiết kế: `SCREEN_SPECIFICATION.md` (system prompt Monaco versioned + diff; editor multi-tab, read-only khi agent giữ lock), `RESPONSIVE_DESIGN.md` (Monaco readOnly cho diff trên màn hình nhỏ; "Canvas/Monaco không khả dụng trên touch nhỏ"), `INTERACTION_DESIGN.md` (phím tắt khi focus Monaco). Chưa có bằng chứng về kích thước bundle thực, cấu hình worker trên Vite 7, hay license các gói ngôn ngữ (Monaco là MIT theo giả định, chưa đọc).

**Khuyến nghị**: dùng `@monaco-editor/react` với `loader.config({ monaco })` để bundle cục bộ; lazy-load theo route; chỉ nạp ngôn ngữ cần thiết.

**Trạng thái**: **Đề xuất**.

---

## 5. Quyết định phụ thuộc chéo

| Phụ thuộc | Mô tả |
|---|---|
| TE-01 ↔ TE-02 | Schema `workflow_runs/node_runs` phải ở Drizzle; code mẫu `prisma.*` trong tài liệu orchestration cần dịch |
| TE-02 ↔ TE-08 | Outbox → Redis → Socket.IO; replay theo `lastEventId` |
| TE-03 ↔ TE-04 ↔ OpenHands (D-006) | Cả ba cùng cắm vào `AgentBrainAdapter`; contract event chung quyết định mapping |
| TE-05 ↔ TE-07 | Phaser đọc Zustand store qua `getState()`/`subscribe` |
| TE-06 ↔ TE-10 | Sidecar Node/native module phụ thuộc cấu hình pnpm/Node |
| TE-09 ↔ TE-11 | Pipeline CI chạy `turbo test`/`playwright`; cache Playwright browsers |

## 6. Rủi ro

| ID | Rủi ro | Mức | Giảm thiểu |
|---|---|---|---|
| TR-1 | Khoá công nghệ dựa trên tài liệu chưa có prototype (ORM, engine, Phaser, Tauri) | Trung bình | Spike có hạn giờ trước khi đóng quyết định tương ứng |
| TR-2 | Số phiên bản/ngày phát hành từ web search (Phaser 4, Drizzle 1.x, Tailwind 4.1, Vite 7…) không được tôi kiểm chứng | Trung bình | Kiểm `npm view` ngay trước khi tạo `package.json` |
| TR-3 | Điều khoản/quota subscription cho dùng OAuth tự động đa session | Cao | Chủ sở hữu xác nhận; thiết kế chuyển sang API key khi cần |
| TR-4 | Codex adapter chưa từng chạy thật | Cao | Live test bắt buộc trước khi cam kết |
| TR-5 | Chọn Turborepo/Recharts mà không có đánh giá | Thấp | Bọc sau abstraction, đổi rẻ |
| TR-6 | Môi trường dev hiện tại (15,77 GB RAM, 6,4 GB trống) giới hạn việc thử nghiệm đồng thời nhiều công cụ | Thấp | Chạy spike tuần tự |

## 7. Vấn đề chưa giải quyết

1. Điều khoản dùng subscription Claude/ChatGPT cho sản phẩm đa-agent (TE-03, TE-04).
2. Có chạy live test Codex sau khi cài CLI không, và khi nào (tốn quota).
3. Vitest cho `apps/api` hay Jest (TE-11) — chờ spike.
4. Có Turborepo ngay từ đầu hay không (TE-09).
5. Thư viện chart cuối cùng và liệu có chuẩn trực quan hoá bắt buộc của tổ chức (TE-12).
6. Cách đóng gói NestJS sidecar (SEA vs `@yao-pkg/pkg` vs ship Node) và xử lý native module (TE-06).
7. Lịch dùng Phaser 4: khi nào nâng từ 3.90 (TE-05).
8. Tailwind v4 + shadcn/ui, Vite 7, React 19.2, TypeScript 5.9 (nằm trong D-001) chưa có mục riêng ở đây vì chưa có tranh chấp; nếu cần phê duyệt riêng, bổ sung.

## 8. Cần người dùng phê duyệt

- [ ] TE-01 Drizzle (và xử lý xung đột Prisma trong tài liệu orchestration).
- [ ] TE-02 Tự xây Conductor thay vì dùng LangGraph/Temporal (cam kết công sức lớn).
- [ ] TE-03 Agent SDK + xác nhận điều khoản sử dụng subscription.
- [ ] TE-04 Cho phép cài Codex CLI và chạy live test (tốn quota); chấp nhận rằng cho tới lúc đó adapter là giả thuyết.
- [ ] TE-06 Lộ trình web trước, Tauri sau; chấp nhận Rust trong đội.
- [ ] TE-09 Turborepo từ đầu hay thêm sau.
- [ ] TE-12 Recharts tạm thời hoặc chỉ định thư viện khác.
- [ ] Các quyết định còn lại (TE-05, 07, 08, 10, 11, 13): duyệt theo lô nếu không có phản đối.
