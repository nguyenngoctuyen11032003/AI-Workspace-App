# CODING STANDARDS — AI WORKSPACE
> **[OWR-3 A.16, QA3 F3-02]** Tên event dạng UPPER_SNAKE trong tài liệu này là tên tương thích. Tên canonical là dotted lowercase; 16 event gốc và ánh xạ đề xuất nằm ở `docs/EVENT_SYSTEM.md` mục 0 (ánh xạ chờ Owner xác nhận, P-16).

Trạng thái: DRAFT (Phase 0). Ngày: 2026-10-30. Tác giả: Delivery/Engineering Lead.
Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].
Liên quan: `DEVELOPMENT_GUIDE.md`, `TESTING.md`, `../SECURITY.md`, `../DATABASE_DESIGN.md`, `../../DECISION_LOG.md` (D-001, D-008, D-014, D-016), `../architecture/WORKSPACE_AND_GIT_ISOLATION.md` §9.

---

## 1. Mục đích

Chuẩn mã cho monorepo TypeScript của AI WORKSPACE, áp dụng cho cả developer người và agent AI, để mã nhất quán, an toàn, kiểm thử được và không vi phạm các ràng buộc cốt lõi: không hardcode agent/workflow, không mock data trong luồng sản phẩm, không credential ở frontend.

## 2. Phạm vi

- Trong: `apps/api` (NestJS), `apps/web` (React 19 + Vite + Tailwind v4 + shadcn/ui), `packages/*`, script `bench/`, cấu hình hạ tầng dạng code (compose, Dockerfile, workflow CI).
- Ngoài: mã do agent sinh cho project của người dùng (chuẩn riêng theo project).
- Phase 0 chưa có mã; các rule dưới đây sẽ được hiện thực thành cấu hình ESLint/tsconfig ở P1-02. [Đã xác minh: chưa có mã]

## 3. Nội dung chi tiết

### 3.1 TypeScript strictness [Đề xuất]

`tsconfig.base.json` (rút gọn):

```jsonc
{
  "compilerOptions": {
    "target": "ES2023", "module": "NodeNext", "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "useUnknownInCatchVariables": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

Quy tắc:

1. Không `any` tường minh. Cần thoát hiểm thì dùng `unknown` + type guard/Zod. Ngoại lệ có comment `// eslint-disable-next-line @typescript-eslint/no-explicit-any -- <lý do + issue>` và được reviewer duyệt.
2. Không `as` ép kiểu để "làm câm" compiler; chỉ cho phép `as const` và ép kiểu sau khi đã validate (đặt trong hàm `parseX`).
3. Không non-null assertion `!` trong mã sản phẩm (cho phép trong test).
4. Dùng discriminated union cho trạng thái và event; `switch` phải exhaustive qua helper `assertNever(x: never)`.
5. Trạng thái domain là union literal sinh từ `packages/contracts` (một nguồn). Ví dụ `type TaskStatus = (typeof TASK_STATUSES)[number]`. Quy ước chữ HOA hay thường cho enum phải theo quyết định cuối của Architect (xem Vấn đề chưa giải quyết 1).
6. ID là branded type (`type TaskId = string & { __brand: 'TaskId' }`) cho các ID chính (TaskId, AgentId, RunId, ProjectId) để không đảo tham số. ID là ULID (`WORKSPACE_AND_GIT_ISOLATION.md` §4). [Đề xuất]
7. Tiền và token: dùng `bigint` hoặc số nguyên đơn vị nhỏ nhất (micro-USD / token count), không `number` thập phân cho tiền. Đơn vị ngân sách chưa chốt (OI-FR-07). [Giả định]
8. Thời gian: lưu/giao tiếp ISO-8601 UTC; trong mã dùng `Date` hoặc `Temporal` thông qua `Clock` inject được.
9. Không `enum` TS (dùng `as const` object + union) để tương thích `isolatedModules` và Zod.
10. Mỗi `package` có `tsconfig.json` riêng dùng project references; `pnpm typecheck` = `tsc -b`.

### 3.2 Lint và format [Đề xuất]

- ESLint flat config + `typescript-eslint` (strict-type-checked), `eslint-plugin-import` (thứ tự, không vòng phụ thuộc), `eslint-plugin-unicorn` (tập con), `eslint-plugin-security`, `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y` (web), `eslint-plugin-tailwindcss` (web, thứ tự class).
- Prettier: `printWidth 100`, `singleQuote`, `trailingComma: all`, `semi: true`. ESLint không chứa rule format.
- `dependency-cruiser` kiểm ranh giới import ở `DEVELOPMENT_GUIDE.md` §3.1 và cấm vòng phụ thuộc.
- Rule tuỳ biến (custom) bắt buộc, gắn với mục 3.8:
  - `aiw/no-hardcoded-role`: cấm chuỗi trùng 18 `role code` seed hoặc tên 18 vai trò xuất hiện trong `if/switch/===` ngoài `seed/` và `test/`.
  - `aiw/no-mock-import-in-src`: cấm import từ `packages/testing`, `**/fixtures/**`, `**/__mocks__/**` trong `src/` của app.
  - `aiw/no-secret-in-web`: cấm trong `apps/web` các identifier/chuỗi khớp `/(api[_-]?key|token|secret|credential|password)/i` được gán giá trị chuỗi literal hoặc đọc từ `import.meta.env` ngoài allowlist (`VITE_API_BASE_URL`, `VITE_WS_URL`).
  - Semgrep: cấm `child_process.exec*`, `shell: true`, `eval`, `new Function`, `dangerouslySetInnerHTML` không qua sanitizer (`SECURITY.md` §9 SAST).
- Lint chạy ở pre-commit (lint-staged) và CI; warning được coi như lỗi trên `main` (`--max-warnings 0`).
- Docs lint: markdownlint cho `docs/**` (CI_CD.md §4.1).

### 3.3 Quy ước đặt tên [Đề xuất]

| Đối tượng | Quy ước | Ví dụ |
|---|---|---|
| File | `kebab-case.ts`; component React `PascalCase.tsx`; test `*.test.ts` | `agent-registry.service.ts`, `AgentChip.tsx` |
| Thư mục | `kebab-case` | `modules/approvals/` |
| Class, interface, type, component | `PascalCase`; không tiền tố `I` | `ProviderAdapter` |
| Hàm, biến | `camelCase`; boolean bắt đầu `is/has/can/should` | `isRunnable` |
| Hằng toàn cục | `SCREAMING_SNAKE_CASE` chỉ cho hằng thật sự bất biến | `MAX_CONCURRENT_HARD_CAP` |
| Event type | `SCREAMING_SNAKE_CASE`, danh từ_quá-khứ-phân-từ, khớp FR-117 | `AGENT_STATE_CHANGED`, `TASK_CREATED` |
| Mã lỗi | `SCREAMING_SNAKE_CASE`, tiền tố miền | `PROVIDER_SESSION_EXPIRED` |
| Bảng DB | `snake_case` số nhiều; cột `snake_case` | `agent_templates.created_at` |
| Route REST | `kebab-case`, danh từ số nhiều, versioned `/api/v1/` | `/api/v1/agents/:agentId/pause` |
| Biến môi trường | `AIW_` + `SCREAMING_SNAKE_CASE` | `AIW_MAX_CONCURRENT_AGENTS` |
| Branch/commit | xem `DEVELOPMENT_GUIDE.md` §3.3, mục 3.9 | |
| Module NestJS | một thư mục = một module = `<name>.module.ts` | `scheduler.module.ts` |
| CSS token | `--ai-accent`, `--status-running`... theo `DESIGN_TOKENS.md` | |

Ngôn ngữ: mã, tên biến, comment kỹ thuật bằng **tiếng Anh**; tài liệu tiếng Việt, thuật ngữ kỹ thuật tiếng Anh. Chuỗi UI nằm trong file i18n (`en` mặc định, `vi` thứ hai theo D-015), không hardcode trong JSX.

### 3.4 Xử lý lỗi [Đề xuất]

1. Lỗi miền dùng lớp `AppError` có `code` (ổn định, dùng cho UI và test), `message` (cho người), `httpStatus`, `retryable: boolean`, `cause?`, `details?` (đã mask). Không `throw new Error('...')` trần trong mã miền; không `throw` chuỗi.
2. Kết quả có thể thất bại **dự đoán được** (policy deny, validation, NOT_SUPPORTED) trả về `Result<T, E>` (hoặc union), không dùng exception cho luồng thường. Exception cho tình huống bất thường.
3. Filter toàn cục của NestJS chuyển `AppError` thành body chuẩn (RFC 9457 problem+json): `{ type, title, status, code, detail, correlationId }`. Không rò stack trace/đường dẫn nội bộ/secret ra client.
4. Adapter phải ánh xạ lỗi CLI sang tập mã chuẩn: `PROVIDER_NOT_INSTALLED`, `PROVIDER_NOT_AUTHENTICATED`, `SESSION_EXPIRED` (AC-028), `NOT_SUPPORTED` (AC-034), `RATE_LIMITED`, `TIMEOUT`, `CANCELLED`, `PROCESS_DIED`.
5. Không nuốt lỗi: `catch {}` rỗng bị cấm. `catch (e)` phải xử lý, log có ngữ cảnh rồi rethrow hoặc chuyển thành `AppError`.
6. Tác vụ nền (BullMQ job) phải phân loại lỗi `retryable` vs `fatal`; retry theo policy (FR-103, FR-288) có backoff + jitter; step phải idempotent (FR-291) bằng idempotency key.
7. Mọi `Promise` phải được `await` hoặc xử lý rõ (`@typescript-eslint/no-floating-promises`).
8. Timeout bắt buộc cho mọi I/O ra ngoài (spawn CLI, HTTP, Docker) bằng `AbortSignal`; không chờ vô hạn (NFR-029, AC-129).
9. Frontend: error boundary theo route, `StateBlock variant="error"` có Retry và `code` + `correlationId` (SCREEN_SPECIFICATION §2).

### 3.5 Logging [Đề xuất]

- Thư viện: `pino` (JSON, cấu trúc), qua `nestjs-pino`. Không `console.*` trong `src` (lint `no-console`, ngoại lệ `bench/`).
- Trường bắt buộc: `ts`, `level`, `msg`, `service`, `correlationId`, và khi có: `projectId`, `taskId`, `agentId`, `executionId`, `providerSessionId` (để khớp bằng chứng T5: task ↔ execution ↔ session).
- Mức: `fatal/error/warn/info/debug/trace`; `info` cho thay đổi trạng thái; `debug` mặc định tắt ở production.
- **Redaction ở tầng logger**: dùng `pino` `redact` + middleware redaction của `packages/policy` (corpus 200 secret, `SECURITY.md` §4.4). Không log: prompt đầy đủ của người dùng ở mức `info`, nội dung file, header `Authorization`, biến môi trường nguyên khối, đường dẫn tới file credential.
- Log của agent (stdout/stderr CLI) được lưu như **artifact** có redaction, không trộn vào log ứng dụng.
- Audit log (hành động nhạy cảm: lệnh, thay đổi file, approval, Emergency Stop) là kênh riêng, append-only, ghi vào DB (`SECURITY.md` §8), không thay bằng log thường.
- Event domain (FR-116) không thay thế log; log không phải nguồn cập nhật UI (FR-118).
- Mỗi request/WS message có `correlationId` (header `x-correlation-id`, sinh nếu thiếu) lan truyền qua job queue và event.

### 3.6 DTO và validation [Đề xuất]

1. **Một nguồn schema**: Zod schema trong `packages/contracts` → sinh type TS, OpenAPI (qua `nestjs-zod`/`zod-to-openapi`) và validator dùng ở cả FE và BE. [Giả định: chọn Zod; Architect xác nhận ở P1-04]
2. Mọi controller dùng `ZodValidationPipe` toàn cục: `whitelist` (loại field lạ), `forbidNonWhitelisted`, `transform`. Request body, query, params và **message WebSocket** đều phải validate.
3. Dữ liệu từ nguồn không tin cậy cũng phải parse: output CLI/stream-json, JSON-RPC của Codex, nội dung file YAML/skill, response của OpenHands. Parse thất bại → `AppError` có mã, không crash.
4. DTO phân biệt `CreateXDto`, `UpdateXDto` (partial), `XResponseDto`. Response DTO **không** chứa field nhạy cảm (credential, đường dẫn tuyệt đối host nhạy cảm, hook token). Test contract kiểm không có field cấm (AC-030).
5. Ràng buộc nghiệp vụ trong schema: `name` bắt buộc và duy nhất trong company (FR-022; unique kiểm ở DB + ánh xạ lỗi 409), độ dài tối đa, regex cho slug, giới hạn kích thước file upload (FR-054), giới hạn mảng.
6. Giá trị `model` chỉ chấp nhận trong tập ListModels của provider (FR-093): validator động ở service, không enum cứng.
7. Version hoá: thay đổi phá vỡ DTO → `/api/v2` hoặc trường `schemaVersion`; event có `schemaVersion`.
8. Phân trang: cursor cho log/event, page-size mặc định 50, tối đa 200.

### 3.7 Kiến trúc mã và module [Đề xuất]

- Backend modular monolith: `controller → service → repository`. Controller chỉ gọi service; service chứa logic; repository truy cập Drizzle. Không để logic nghiệp vụ trong controller hay repository.
- Transaction: thay đổi trạng thái domain + ghi event vào **outbox** trong cùng một transaction (D-014); không `publish` trực tiếp sau commit.
- Mọi chuyển trạng thái agent/task đi qua **state machine** một nơi; không `UPDATE ... SET status` rải rác (FR-025).
- Dependency injection: không singleton toàn cục ẩn; `Clock`, `IdGenerator`, `ProcessSpawner`, `ProviderAdapter` được inject để test được.
- Spawn tiến trình: chỉ `spawn(file, args[])`, không shell; qua `ProcessSpawner` tập trung để áp dụng Job Object/timeouts/stagger (RESOURCE_MANAGEMENT).
- Frontend: `features/<name>/{api,components,hooks,store}`; server state bằng TanStack Query; UI state cục bộ bằng Zustand; cập nhật realtime bằng event → cập nhật cache. Không `fetch` trực tiếp trong component. Component tuân theo `COMPONENT_SPECIFICATION.md`.
- Virtual Office: Phaser scene chỉ nhận lệnh qua event bus từ Zustand; không có timer tự sinh hành vi (FR-160).
- Không dùng thư viện copy mã từ My Virtual Office (AGPL); mọi mã tái sử dụng từ ChatDev/Pixel Agents/OpenHands phải có attribution trong `THIRD_PARTY_NOTICES.md` (D-012). [Đề xuất]

### 3.8 Pattern bị cấm (forbidden patterns) [Đề xuất — Cần người dùng phê duyệt]

Vi phạm là lý do từ chối PR. Cột "Cách phát hiện" là cơ chế tự động hoá.

| # | Cấm | Vì sao | Cách phát hiện |
|---|---|---|---|
| F-01 | **Hardcode danh sách agent/role** (mảng 18 role, `if (role === 'backend')`, `switch(role)`) ngoài `seed/` | D-016, FR-043, `AGENT_ORGANIZATION.md` nguyên tắc 1: role là dữ liệu | `aiw/no-hardcoded-role`, review |
| F-02 | **Hardcode workflow** (chuỗi 8 stage SDLC dưới dạng if/else trong code) thay vì đọc template/DAG từ DB | FR-114, FR-119: workflow là template dữ liệu; D-003 | grep tên stage trong `src/`, review |
| F-03 | **Mock/fake/mẫu dữ liệu trong luồng sản phẩm**: dữ liệu "Lorem" trong UI, mảng agent giả để "trông có việc", `setInterval` giả lập tiến độ | Yêu cầu "chỉ event thật" (FR-160, FR-118) | `aiw/no-mock-import-in-src`, kiểm bundle, review |
| F-04 | Credential ở frontend: token/API key trong mã, `localStorage`, biến `VITE_*`, payload API | FR-087, AC-030, D-008 | `aiw/no-secret-in-web`, E2E quét payload |
| F-05 | Đọc `.credentials.json`/`auth.json` của CLI hoặc trích token | FR-085, AC-031 | Semgrep rule |
| F-06 | Lưu secret plaintext trong Postgres/file cấu hình; commit `.env` | FR-086, D-008 | gitleaks, schema review |
| F-07 | `child_process.exec`, `shell: true`, nối chuỗi lệnh | Command injection | Semgrep |
| F-08 | Cập nhật UI/office từ nguồn khác event (poll DB trực tiếp, timer) | FR-118, AC-059 | review, test "tắt event bus" |
| F-09 | Đổi trạng thái entity ngoài state machine/event | FR-025 | `no-restricted-syntax` với `.update({ status` ngoài service chuyên trách |
| F-10 | Ghi file ngoài vùng `owns` hoặc sửa worktree của task khác | D-009, FR-039 | hook + diff validation |
| F-11 | Magic number cho limit (concurrency, timeout, budget) | RESOURCE_MANAGEMENT §10 cấu hình | ESLint `no-magic-numbers` (ngoại lệ 0,1,-1,100,1000), review |
| F-12 | Auto-deploy project sinh ra lên production | FR-126, DEPLOYMENT §9 | review, test |
| F-13 | `any`, `@ts-ignore`, `@ts-nocheck` | An toàn kiểu | ESLint (`@ts-expect-error` kèm mô tả được phép) |
| F-14 | Bypass hook/approval (`--dangerously-skip-permissions`, `bypassPermissions`) trong adapter | `SECURITY.md` R8 | snapshot test `--settings` |
| F-15 | Sao chép mã/prompt/layout/asset từ My Virtual Office (AGPL-3.0) | D-012 | review, `LICENSE_AND_REUSE.md` checklist |
| F-16 | Asset có giấy phép không rõ hoặc không thuộc CC0/MIT/OFL | D-012 | `assets-manifest.json` bắt buộc + CI kiểm |
| F-17 | `console.log`, in biến môi trường, log prompt đầy đủ | Rò rỉ | ESLint `no-console`, redaction test |
| F-18 | Test pass chỉ vì mock thay luồng thật; `it.skip`/`.only` còn sót | `TESTING.md` §3.5 | ESLint `no-only-tests`, quy tắc nhãn bằng chứng |
| F-19 | Hiển thị số liệu giả/ước tính như thật (cost, usage) mà không ghi nhãn "estimated" | FR-096 | review UI, `UsageValue` component có prop `estimated` |
| F-20 | Chuỗi UI hardcode trong JSX (không qua i18n) | D-015 | `eslint-plugin-i18next` [Đề xuất] |

Ngoại lệ hợp lệ cho F-01/F-02: dữ liệu `seed/*.yaml|json`, test, tài liệu ví dụ. Seed là **template**, khi seed vào DB mang cờ `isSeed=true`, người dùng có thể archive/clone.

### 3.9 Commit và PR [Đề xuất]

Định dạng Conventional Commits (đồng bộ `WORKSPACE_AND_GIT_ISOLATION.md` §9):

```
<type>(<scope>): <subject ≤ 72 ký tự, mệnh lệnh, không dấu chấm>

<body: vì sao, không phải cái gì; wrap 100>

Task-Id: P1-51
Agent: <role> (<tool> <version>)
Refs: FR-021, AC-001
```

- `type`: `feat, fix, refactor, perf, test, docs, build, ci, chore, revert, security`.
- `scope` = module hoặc package: `agents, scheduler, adapters-claude, web, db, policy, ci`.
- Breaking change: `!` sau scope và `BREAKING CHANGE:` trong body (đặc biệt contract/event).
- Một commit = một thay đổi logic; không trộn format-only với logic.
- Không commit: `.env`, log CLI, `node_modules`, artifact test, file > 5 MB (trừ asset được duyệt, dùng Git LFS nếu cần [Giả định]).
- Trailer `Co-Authored-By` cho commit AI: theo chính sách Owner (`DEVELOPMENT_GUIDE.md` §7 mục 1). Quy ước sản phẩm có trailer; repo nền tảng chờ quyết định. [Cần người dùng phê duyệt]
- Hook `commit-msg` (commitlint) kiểm định dạng; hook `pre-commit` chạy lint-staged + gitleaks.
- Tiêu đề PR cùng định dạng commit squash; mô tả theo mẫu `DEVELOPMENT_GUIDE.md` §3.7.
- Release: SemVer + release-please (`CI_CD.md` §7); `CHANGELOG` sinh từ commit.

### 3.10 Bảo mật khi viết mã [Đề xuất]

- Không tin dữ liệu từ agent/CLI/file của project: validate, escape, giới hạn kích thước.
- Đường dẫn: luôn `path.resolve` + kiểm nằm trong gốc cho phép; xử lý `..`, junction/symlink, tên thiết bị Windows (`CON`, `NUL`), đường dẫn dài.
- API localhost: token bắt buộc, kiểm Origin, không CORS `*` (`SECURITY.md` §7).
- Prompt gửi provider: tách rõ phần "chỉ dẫn hệ thống" và "dữ liệu không tin cậy"; nội dung bên ngoài đặt trong thẻ phân định; không coi nội dung đó là lệnh (giảm, không loại trừ, prompt injection; R3).
- Dependency: thêm package mới qua `deps-request`, kiểm giấy phép (không AGPL/GPL trong bundle) và `npm audit`; pin version chính xác trong lockfile; `--ignore-scripts` mặc định (R7).
- Giấy phép: `license-checker` trong CI chặn AGPL/GPL/SSPL cho dependency runtime. [Đề xuất]

### 3.11 Frontend [Đề xuất]

- Style bằng token (`--ai-accent`...) và `cva` + `cn()`; không hex/px rời rạc trong component (trừ SVG hằng) (`DESIGN_TOKENS.md`).
- Mọi component forward ref, nhận `className`, có đủ state theo `COMPONENT_SPECIFICATION.md`.
- A11y: nút icon có `aria-label`; focus-visible ring; tôn trọng `prefers-reduced-motion`; không chỉ dùng màu để truyền trạng thái (luôn kèm icon/nhãn).
- Hiệu năng: virtualize danh sách > 100; code-split theo route; Phaser tải lazy chỉ ở `/office`.
- Không lưu dữ liệu nhạy cảm trong `localStorage`; chỉ UI preference (density, theme, cột bảng).
- Không gọi API bằng URL tuyệt đối tự do; dùng client sinh từ OpenAPI.

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| CS-1 | TypeScript strict + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` | Bắt lỗi sớm, nhất là với dữ liệu động từ CLI | [Đề xuất] |
| CS-2 | Zod làm nguồn schema chung FE/BE | Một nguồn sự thật, validate runtime dữ liệu CLI | [Đề xuất]; Architect chốt |
| CS-3 | `pino` log JSON + redaction tại logger | Yêu cầu FR-090, NFR bảo mật | [Đề xuất] |
| CS-4 | Rule ESLint tuỳ biến cho F-01/F-03/F-04 | Biến yêu cầu cốt lõi thành kiểm tra tự động, không phụ thuộc trí nhớ của agent | [Đề xuất] |
| CS-5 | Outbox + state machine duy nhất | Nhất quán event và trạng thái (D-014, FR-025) | [Đề xuất] |
| CS-6 | Mã tiếng Anh, tài liệu tiếng Việt | Công cụ/agent/OSS phổ biến; yêu cầu tài liệu của Owner | [Đề xuất] |
| CS-7 | Conventional Commits + trailer Task-Id | Truy vết task ↔ commit, khớp thiết kế sản phẩm | [Đề xuất] |

## 5. Dependencies

- `packages/contracts` (Zod schema), `packages/events` (catalogue), `packages/policy` (redaction, normalizer).
- Công cụ: ESLint, Prettier, commitlint, lint-staged, gitleaks, Semgrep, dependency-cruiser, license-checker.
- Quyết định D-001 (stack), D-002 (Drizzle), D-014 (outbox) được duyệt.
- `DESIGN_TOKENS.md` để rule CSS.

## 6. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Rule quá chặt làm agent mất thời gian sửa lint | Giảm năng suất | Cấu hình sớm ở P1-02, dạy qua `AGENTS.md`, `--fix` tự động |
| Rule `no-hardcoded-role` dương tính giả (từ "frontend" trong mô tả) | Gây nhiễu | Chỉ kiểm literal trong biểu thức điều kiện/so sánh, danh sách ngoại lệ |
| Zod ↔ OpenAPI sinh lệch | Contract sai | Test snapshot OpenAPI + contract test |
| Pino redaction bỏ sót định dạng secret mới | Rò rỉ log | Corpus mở rộng, gitleaks quét artifact |
| `exactOptionalPropertyTypes` gây ma sát với thư viện | Chậm | Cho phép tắt theo package có lý do ghi trong tsconfig |

## 7. Vấn đề chưa giải quyết

1. Quy ước chữ HOA vs thường cho enum `task_status` và enum khác (`DECISION_LOG.md` mục 5; `DATABASE_DESIGN.md` mục 12). Chờ Architect.
2. 12 trạng thái agent (FR-024) và danh sách trạng thái trong `SCREEN_SPECIFICATION.md` (idle, starting, working, waiting_input...) chưa đồng nhất; `StatusBadge` trong DESIGN_SYSTEM dùng bộ 8 trạng thái khác nữa. Cần bảng ánh xạ chính thức.
3. [Cần người dùng phê duyệt] Chính sách `Co-Authored-By` cho repo nền tảng.
4. Chọn Zod hay lựa chọn khác (Valibot, class-validator): Zod là giả định.
5. Đơn vị tiền/ngân sách (OI-FR-07) ảnh hưởng kiểu dữ liệu 3.1 mục 7.
6. Có dùng `eslint-plugin-i18next` ngay Phase 1 hay sau khi có bộ chuỗi (VI là locale thứ hai).
7. Git LFS cho asset pixel-art: chưa quyết.

## 8. Acceptance Criteria (cho chuẩn mã)

- [ ] `pnpm lint --max-warnings 0` và `pnpm typecheck` xanh trên repo trống Phase 1 (P1-02) và chặn merge khi đỏ.
- [ ] Mỗi rule F-01, F-03, F-04, F-07 có test âm (mã vi phạm mẫu làm lint fail) trong `packages/eslint-config`.
- [ ] Test bundle chứng minh `apps/web` không chứa chuỗi khớp secret pattern và không chứa FakeProvider.
- [ ] Mọi controller dùng validation pipe toàn cục; test: body có field lạ → 400.
- [ ] Log mẫu chứa secret giả bị mask (corpus redaction đạt 0 leak).
- [ ] commitlint chặn commit sai định dạng; PR mẫu có trailer `Task-Id`.
- [ ] `THIRD_PARTY_NOTICES.md` và `assets-manifest.json` tồn tại trước khi đưa asset hoặc mã tái sử dụng đầu tiên vào repo.
