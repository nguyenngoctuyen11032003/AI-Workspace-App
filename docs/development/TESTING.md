# TESTING — Chiến lược kiểm thử AI WORKSPACE

Trạng thái: DRAFT (Phase 0). Ngày: 2026-10-30. Tác giả: Delivery/Engineering Lead.
Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].
Liên quan: `CI_CD.md`, `DEVELOPMENT_GUIDE.md`, `CODING_STANDARDS.md`, `../product/ACCEPTANCE_CRITERIA.md`, `../research/MULTI_AGENT_SCALABILITY.md` §9, `../SECURITY.md` §9, `../IMPLEMENTATION_ROADMAP.md`.

---

## 1. Mục đích

Định nghĩa cách chứng minh rằng AI WORKSPACE hoạt động thật: tầng kiểm thử, công cụ, quy tắc dùng CLI thật so với mock trong CI, bộ test chứng minh 18 agent đồng thời (T1-T7), test bảo mật, và mục tiêu coverage. Gắn mọi test với Acceptance Criteria (AC-001…AC-130) để truy vết.

## 2. Phạm vi

- Trong: test của nền tảng (backend NestJS, frontend React, Phaser office, adapters, policy, sandbox, workflow engine, scheduler).
- Ngoài: test của project do nền tảng sinh ra cho người dùng (do agent `qa`/`test_automation` tạo trong workspace của project đó).
- Phase 0 chưa có mã; mọi lệnh/đường dẫn là thiết kế Phase 1. [Đã xác minh]

## 3. Nội dung chi tiết

### 3.1 Nguyên tắc cốt lõi [Đề xuất]

1. **Bằng chứng trước tuyên bố.** Một tính năng không "hoàn thành" nếu chỉ có test mock (xem 3.5 quy tắc vàng).
2. **Mock không thay thế luồng sản phẩm.** Mock/fake chỉ dùng để cô lập nhiễu (mạng, quota, thời gian) trong CI; mọi luồng người dùng cuối phải có ít nhất một test chạy bằng thành phần thật (DB thật, Redis thật, CLI thật khi có thể).
3. **Test pyramid** nhiều unit, vừa integration, ít nhưng chắc E2E.
4. **Test xác định (deterministic).** Đồng hồ, UUID/ULID, random được inject; cấm `setTimeout` thực trong unit test (dùng fake timers).
5. **Mỗi AC có ít nhất một test ánh xạ** (bảng 3.9); AC chưa có test thì ghi rõ.
6. **Nhãn trạng thái bằng chứng:** `VERIFIED-REAL` (chạy CLI/DB thật, có log), `VERIFIED-MOCK` (chỉ chạy với mock), `UNVERIFIED`.

### 3.2 Test pyramid và công cụ [Đề xuất]

| Tầng | Công cụ | Phạm vi | Hạ tầng | Chạy ở đâu | Tỷ trọng mục tiêu |
|---|---|---|---|---|---|
| Static | ESLint, `tsc --noEmit`, Semgrep, gitleaks, dependency-cruiser | toàn repo | không | pre-commit + CI `lint`, `typecheck`, `security` | — |
| Unit | Vitest | hàm thuần, reducer, policy, state machine, DTO schema, React component logic | không (không I/O) | mọi PR | ~65% |
| Component | Vitest + React Testing Library + `jest-axe` | component trong `COMPONENT_SPECIFICATION.md` | jsdom | mọi PR | trong 65% trên |
| Integration | Vitest + Testcontainers (Postgres 16, Redis 7) hoặc compose service của CI | repository, outbox, BullMQ, scheduler semaphore Lua, WS gateway, migration | Postgres/Redis thật | mọi PR chạm `apps/api`/`packages/db` | ~25% |
| Contract | Vitest + `openapi` schema diff + event schema snapshot | `packages/contracts`, `packages/events` | không | mọi PR | trong 25% |
| E2E UI | Playwright (Chromium; WebKit/Firefox nightly) | luồng người dùng trên app đã build | api + web + DB thật, provider fake hoặc thật tuỳ job | `e2e` job sau `build` | ~7% |
| Provider real | Vitest tag `@real-provider` | adapter với `claude`/`codex` CLI thật | máy dev có OAuth login | tay / nightly self-hosted | ~2% |
| Security | Vitest, script container, Semgrep, gitleaks, trivy | `SECURITY.md` §9 | Docker | mọi PR chạm vùng nhạy cảm + nightly | cắt ngang |
| Performance / concurrency | script `bench/`, k6 (tuỳ chọn) | T1-T7, FPS office | máy tham chiếu | tay, cần phê duyệt quota | ~1% |

Ghi chú: tỷ trọng là hướng dẫn số lượng test, không phải KPI. [Giả định]

### 3.3 Quy ước đặt tên, tag và script [Đề xuất]

| Mục | Quy ước |
|---|---|
| Tên file | `*.test.ts(x)` (unit/component/integration), `*.e2e.ts` (Playwright), `*.real.test.ts` (provider thật) |
| Tag Vitest | `@unit`, `@int`, `@real-provider`, `@security`, `@perf`, `@flaky` (gắn trong tên `describe`) |
| Tên test | `AC-0xx: <hành vi>` khi test chứng minh một AC, để truy vết bằng grep |
| Script | `pnpm test` (unit+component), `test:int`, `test:e2e`, `test:security`, `test:provider:real`, `bench:concurrency` |
| Artifact | `artifacts/test/<run-id>/` (log, trace, JSON); không commit, trừ evidence được duyệt ở `docs/evidence/` |

### 3.4 Quy ước từng tầng [Đề xuất]

**Unit (Vitest).** File `*.test.ts` đặt cạnh nguồn; `describe` theo đơn vị, `it` mô tả hành vi. Không I/O, không import `apps/api` khác module. Dùng `vi.useFakeTimers()`. State machine (agent 12 state FR-024, task 10 state FR-108) kiểm bằng bảng chuyển hợp lệ/không hợp lệ (property test với `fast-check`).

**Component.** Mỗi component trong catalogue có test: render các variant, các state (loading/empty/error/disabled), sự kiện, và `expect(await axe(container)).toHaveNoViolations()`. Không snapshot HTML lớn; chỉ snapshot token/class của variant (`cva`).

**Integration.** Dùng DB thật: mỗi file test tạo schema riêng hoặc transaction rollback; migration chạy từ đầu. Kiểm: outbox → Redis pub/sub (D-014), `lastEventId` replay, semaphore Lua global + per-provider (D-003), workflow `WAITING_HUMAN` không giữ worker, crash recovery rebuild semaphore từ Postgres.

**Contract.** (a) OpenAPI sinh từ NestJS so với snapshot đã commit; thay đổi phá vỡ phải nâng version; (b) mỗi event trong catalogue có JSON schema và fixture hợp lệ/không hợp lệ; (c) FE test dùng cùng fixture với BE (một nguồn trong `packages/testing`).

**E2E (Playwright).** Page Object theo màn hình (`SCREEN_SPECIFICATION.md`). Selector dùng `getByRole`/`data-testid`, không CSS dài. Trace + video khi fail, lưu artifact CI. Luồng bắt buộc (mỗi luồng ánh xạ AC):

| E2E | Nội dung | AC |
|---|---|---|
| E2E-01 | Connect provider → trạng thái Connected/Not installed/Not authenticated | AC-021, 023, 024 |
| E2E-02 | Agent Builder tạo agent, validation, model từ ListModels | AC-001…003 |
| E2E-03 | Tạo project từ prompt → SDLC stage 1-2 → Approval Center → Approve/Reject | AC-041…046 |
| E2E-04 | Task Board realtime theo event, tắt event bus thì UI không đổi | AC-059, AC-063 |
| E2E-05 | Emergency Stop hiển thị mọi màn và dừng session | AC-109, AC-112 |
| E2E-06 | Virtual Office hiển thị 14 phòng, nhân vật phản ứng event thật | AC-086…093 |
| E2E-07 | Kill app giữa Stage 5, mở lại, resume từ checkpoint | AC-121, AC-122 |
| E2E-08 | Skill Library + Teach Agent (rule review trước khi lưu) | AC-071…076 |

### 3.5 Provider tests: CLI thật và mock trong CI [Đề xuất]

Hiện trạng [Đã xác minh]: Claude Code CLI 2.1.312 đã cài và đăng nhập (claude.ai OAuth, gói team); **Codex CLI chưa cài**; Docker Desktop đã cài nhưng daemon chưa chạy lúc kiểm tra.

| Lớp | Mô tả | Chạy ở | Nhãn bằng chứng |
|---|---|---|---|
| P-A Adapter conformance (fake) | Bộ test chung cho 13 operation của `ProviderAdapter` (FR-081) chạy với **FakeProvider** phát stream-json ghi sẵn | CI mọi PR | VERIFIED-MOCK |
| P-B Parser/mapper | Parse stream-json/JSON-RPC ghi từ CLI thật (fixture "golden" có nguồn gốc: ghi lại bằng lệnh thật, kèm version CLI + ngày) | CI mọi PR | VERIFIED-MOCK (golden từ thật) |
| P-C Claude real smoke | `claude -p` qua adapter: Connect, ListModels, SendMessage, StreamEvents, CancelTask ≤ 5 s (AC-026), ResumeSession, thông báo rate-limit thật | máy dev có login; nightly self-hosted runner nếu Owner chấp thuận | VERIFIED-REAL |
| P-D Codex real smoke | tương tự P-C qua `codex app-server`, dự phòng `codex exec --json` (D-005) | **chặn tới khi Owner cài Codex** | UNVERIFIED cho đến lúc đó |
| P-E Provider lifecycle | CLI chưa cài / chưa login / session hết hạn / kill giữa chừng (AC-023, 024, 028, 124, 130) bằng cách chạy adapter với binary giả có exit code tương ứng | CI | VERIFIED-MOCK |

**Quy tắc: mock không bao giờ thay thế luồng sản phẩm** [Đề xuất — Cần người dùng phê duyệt]:

1. FakeProvider **chỉ** nằm trong `packages/testing` và `apps/*/test/**`; bundle production không import được (lint `no-restricted-imports` + kiểm tra bundle bằng `grep`/size-limit trong CI).
2. Biến môi trường chọn provider (`AIW_PROVIDER_MODE=fake`) chỉ được chấp nhận khi `NODE_ENV=test`; ở `development`/`production` backend từ chối khởi động nếu gặp giá trị này.
3. Không dữ liệu mẫu (agent, task, workflow) nào được seed trong chế độ production: dữ liệu seed là **template** (18 AgentTemplate, skill, workflow template) chứ không phải "agent giả đang chạy" (FR-160, D-016).
4. Test harness mô phỏng event cho Virtual Office (AC-096) tách khỏi production, chạy qua route/flag chỉ có trong build test.
5. Một AC thuộc nhóm B (Provider), C (Workflow), F (Security) **không được** đánh dấu đạt chỉ dựa trên `VERIFIED-MOCK`; cần ít nhất một lần `VERIFIED-REAL` có log lưu ở `docs/evidence/` hoặc đính kèm PR. Ngoại lệ: AC phụ thuộc Codex ở trạng thái `UNVERIFIED` tới khi CLI cài.
6. Khi CI chạy E2E với fake provider, báo cáo CI phải ghi rõ "provider=fake" ở tiêu đề job để không bị nhầm là bằng chứng thật.
7. Fixture golden phải có header: lệnh sinh, version CLI, ngày, và đã redact secret.

Chi phí quota: test thật tốn quota subscription dùng chung (một lệnh `claude -p` nhẹ ≈ 48k token tiền xử lý, ≈ 0.37 USD giá niêm yết [Đã xác minh, `MULTI_AGENT_SCALABILITY.md` §4.2]). Do đó P-C giới hạn ≤ 10 lần gọi mỗi lượt chạy, và không chạy trên mỗi PR. [Đề xuất]

### 3.6 Bộ test chứng minh đồng thời 18 agent (T0-T7) [Đề xuất] — [Cần người dùng phê duyệt trước khi chạy]

Nguồn: `MULTI_AGENT_SCALABILITY.md` §9 (bản gốc) và §14 (tiêu chí). Yêu cầu của Owner: bằng chứng gồm **18 session_id thật, 18 task id, log, số đo RAM/CPU, không rate-limit**. [Đã xác minh]

| Bước | Nội dung | Evidence | Điều kiện chạy | Ánh xạ AC |
|---|---|---|---|---|
| T0 | Chuẩn bị: config dir tối giản, 18 workspace `bench/ws-01..18`, script `bench/run-concurrency.ps1 -N`, prompt nhẹ ("Reply OK") và trung bình (đọc file, tóm tắt 5 dòng) | script + commit | Owner đồng ý chạy | — |
| T1 | N=5 nhẹ × 3 lần | 15 JSON (session_id, cost, usage), bảng RSS/CPU, wall time | quota đủ | AC-063 |
| T2 | N=10 nhẹ × 2 lần | như T1 | T1 pass | — |
| T3 | N=18 nhẹ × 2 lần, stagger tối đa 3 spawn / 2 s | như T1 + `Get-Process` mỗi 1 s | T2 pass | AC-064, AC-095 (phần hệ thống) |
| T4 | N=18 trung bình (đọc file, ~3 turn) | so `duration_api_ms` N=18 với N=1 | T3 pass | AC-064 |
| T5 | Qua Agent Scheduler thật (Phase 3+): 18 task READY, limit 18; ghi `task.id`, `execution.id`, `providerSessionId`, `scheduler_event`, log file | export DB + log | scheduler hoàn thành | AC-048, AC-063, AC-064 |
| T6 | Chaos: kill ngẫu nhiên 3 process giữa T5 | `scheduler_event REASSIGN`, không spawn đôi | T5 pass | AC-124, AC-130 |
| T7 | Báo cáo `docs/performance/CONCURRENCY_BENCH_<date>.md`: bảng, link JSON, version CLI, ngày, gói subscription, pass/fail | tài liệu | T1-T6 | tổng hợp |

Tiêu chí đạt (trích `MULTI_AGENT_SCALABILITY.md` §14):

- T3: 0 lỗi, 0 rate-limit, 18 session_id khác nhau mỗi lần; tổng RSS đỉnh < 70% RAM free tại thời điểm chạy; wall time < 3× N=1.
- T4: `duration_api_ms` trung bình N=18 ≤ 2× N=1.
- T5: 18 `task.id` ↔ 18 `execution.id` ↔ 18 `providerSessionId` khớp giữa DB và log.
- T6: sau chaos, `COUNT(RUNNING)` = giá trị semaphore; task bị kill được cấp lại với `attempt+1`.
- Bất kỳ tiêu chí nào fail: max hiệu dụng = N lớn nhất đã pass, ghi vào `RESOURCE_MANAGEMENT.md` §10 (D-010: chỉ nâng 5 → 10 → 18 khi có bằng chứng).

Dự toán: T1-T4 ≈ 89 spawn nhẹ ≈ 33 USD list ở spawn tax hiện tại (≈ 11 USD nếu tối giản) [Đề xuất, trích nguồn]. Chạy đầu cửa sổ quota; dừng ngay khi thấy `RATE_LIMIT`.

Phân biệt: T1-T4 là benchmark process CLI thô (không cần app); T5-T6 là test hệ thống qua app. Kết luận "18 khả thi" cần cả hai. T5-T6 phụ thuộc Phase 3 (scheduler) và Phase 4 (worktree isolation), nên nằm trong exit criteria Phase 7 (`IMPLEMENTATION_ROADMAP.md`). Hạn chế: kết quả chỉ chứng minh trên máy tham chiếu hiện tại (i5-11400H, 16 GB, ~6.4 GB free lúc đo) [Đã xác minh].

### 3.7 Security tests [Đề xuất]

Nguồn: `SECURITY.md` §9. Rút gọn và ánh xạ AC:

| Nhóm | Công cụ | Nội dung | AC |
|---|---|---|---|
| Denylist/normalizer | Vitest + `fast-check` | ≥ 300 case (mỗi pattern ≥ 3 dương + 3 âm), corpus obfuscation, fuzz 10k lệnh | AC-103 |
| Hook integration | Vitest + `claude -p` thật (nightly) hoặc mock stream-json | prompt chạy lệnh bị cấm → `policy_deny` trong audit | AC-103, AC-104, AC-111 |
| Container config | Vitest + `docker inspect` | `Privileged=false`, `CapDrop` có ALL, `no-new-privileges`, `ReadonlyRootfs`, giới hạn Memory/Pids | AC-101, AC-102 |
| Egress | script trong container | `curl` ra ngoài fail khi network deny; chỉ registry khi phase install | AC-106 |
| Filesystem | integration | đọc ngoài workspace fail; project A không chạm project B | AC-105, AC-107 |
| Secret scan | gitleaks (pre-commit + CI), trufflehog nightly | repo + diff agent + DB + log + payload | AC-029, AC-113 |
| Redaction | Vitest | corpus 200 secret × encoding (raw, JSON-escaped, base64, URL, chunk tách) → 0 leak; < 1 ms/KB | FR-090, AC-029 |
| Frontend payload | Playwright + `page.on('response')` | quét mọi payload HTTP/WS tới FE bằng regex secret | AC-030 |
| Adapter code review | Semgrep custom rule | cấm đọc `.credentials.json`, `auth.json` | AC-031 |
| API security | supertest + ws | không token → 401; sai Origin → 403; token trong query → 401; WS không auth trong 2 s → 4401 | NFR bảo mật |
| Manifest enforcement | Vitest | ghi ngoài vùng sở hữu → ask/deny; symlink/junction ra ngoài → deny; diff validation bắt `.env` mới | AC-011 |
| Emergency Stop | integration + chaos | ≤ 5 s: mọi session cancel, container dừng | AC-109, AC-110 |
| Dependency hygiene | `npm audit`, osv-scanner, lockfile-lint | fail khi high/critical | — |
| SAST | Semgrep | cấm `child_process.exec` nối chuỗi, cấm `shell: true`; chỉ `spawn(file, args[])` | — |
| Prompt injection | Vitest + corpus | nội dung file/web độc hại không tạo hành động ngoài policy; kiểm hành động bị chặn, không kiểm "mô hình không bị lừa" | R3 trong `SECURITY.md` |

Gate: PR chạm `policy/`, `hooks/`, `container/`, `redaction/` cần 100% test pass, review `security`, coverage ≥ 90% (trích `SECURITY.md`). Các test container **không chạy được khi Docker daemon tắt** [Đã xác minh: daemon chưa chạy lúc kiểm tra]; khi đó job `security-container` được đánh `SKIPPED-NO-DOCKER` và AC-101/102/105/106 giữ trạng thái `UNVERIFIED` (AC-114 kiểm hành vi cảnh báo).

### 3.8 Mục tiêu coverage [Đề xuất]

| Phạm vi | Line | Branch | Ghi chú |
|---|---|---|---|
| `packages/policy`, `redaction`, `hooks`, `container` | ≥ 90% | ≥ 85% | gate bảo mật |
| State machine, scheduler, workflow engine, outbox | ≥ 85% | ≥ 80% | logic lõi |
| `apps/api` service layer còn lại | ≥ 75% | ≥ 65% | |
| `packages/adapters-*` | ≥ 80% (với fake) | ≥ 70% | cộng P-C/P-D khi có |
| `apps/web` logic/hook/store | ≥ 70% | ≥ 60% | |
| Component UI | ≥ 60% | — | quan trọng hơn là axe + state |
| Phaser office | logic (FSM, pathfinding, event→animation mapping) ≥ 80%; render không đo | | |
| Tổng repo | ≥ 75% | ≥ 65% | sàn, không phải đích |

Quy tắc: coverage không giảm so với `main` (diff coverage ≥ 80% cho dòng mới). Coverage là chỉ báo phụ; test có ý nghĩa (assert hành vi) quan trọng hơn. Mutation testing (Stryker) cho `policy` và `scheduler` là tuỳ chọn Phase 6. Mọi con số là [Đề xuất — Cần người dùng phê duyệt].

### 3.9 Ma trận AC → loại test (tóm tắt) [Đề xuất]

| Nhóm AC | Unit | Integration | E2E | Real provider | Ghi chú |
|---|---|---|---|---|---|
| A Agent (001-015) | state machine, validation | CRUD, ownership | E2E-02 | AC-004, 009 cần thật | AC-011 dùng hook thật |
| B Provider (021-034) | parser, error map | adapter lifecycle | E2E-01 | bắt buộc (Claude); Codex UNVERIFIED | AC-029…032 là test bảo mật |
| C Workflow (041-064) | DAG, retry, scoring | engine + Redis + Postgres | E2E-03/04/07 | ít nhất một SDLC thật nhỏ | AC-064 xem 3.6 |
| D Skill (071-082) | lint, relevance | projection layer | E2E-08 | AC-078, 079 cần thật | |
| E Office (086-096) | FSM, mapping event | event replay | E2E-06 | không cần | FPS đo trên máy tham chiếu |
| F Security (101-114) | policy | container, egress | E2E-05 | hook thật | cần Docker chạy |
| G Reliability (121-130) | — | recovery, heartbeat | E2E-07 | một lần thật | AC-127 lặp 20 lần |

### 3.10 Dữ liệu và môi trường test [Đề xuất]

- Factory (`packages/testing`) sinh dữ liệu hợp lệ theo schema; không dùng dữ liệu đời thật.
- DB test tách khỏi DB dev (`aiw_test`), cổng bind `127.0.0.1`.
- CI: GitHub-hosted `ubuntu-24.04` có Docker; thêm `windows-2025` cho job `unit` để bắt lỗi path/shell Windows [Đề xuất]. Sandbox/Windows job-object thật chỉ kiểm được trên Windows, nên test Windows-specific chạy ở runner Windows hoặc máy dev.
- Flaky test: gắn `@flaky` có issue, retry tối đa 1 lần; quá 3 lần/tuần phải sửa hoặc xoá.
- Thời gian mục tiêu CI: unit+lint+typecheck ≤ 5 phút, integration ≤ 10 phút, e2e ≤ 15 phút. [Giả định]

### 3.11 Kiểm thử UI/UX và hiệu năng [Đề xuất]

- Accessibility: `jest-axe` ở component, `@axe-core/playwright` ở E2E; mục tiêu WCAG 2.2 AA theo `DESIGN_SYSTEM.md`.
- Visual regression: Playwright screenshot cho 5 màn Core (Dashboard, Agent Detail, Task Board, Approval Center, Workflow Builder) ở light/dark; ngưỡng diff 0.2%. Không dùng cho Virtual Office (canvas động).
- FPS Virtual Office (AC-087 ≥ 30 FPS, AC-095 18 nhân vật): đo bằng `requestAnimationFrame` sampler trong Playwright trên máy tham chiếu; kết quả ghi kèm cấu hình máy.
- Responsive: chạy E2E smoke ở 3 viewport (`RESPONSIVE_DESIGN.md`).

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| TS-1 | Vitest + Playwright | Cùng hệ TypeScript/Vite, nhanh, Playwright có trace | [Đề xuất] |
| TS-2 | Mock chỉ cô lập, không thay luồng sản phẩm; nhãn VERIFIED-REAL/MOCK | Ngăn "xanh giả"; đáp ứng yêu cầu Owner về bằng chứng thật | [Cần người dùng phê duyệt] |
| TS-3 | Provider real test chạy tay/nightly, giới hạn số lần | Quota dùng chung, tốn tiền quy đổi | [Đề xuất] |
| TS-4 | T1-T7 tách benchmark CLI thô và test hệ thống | Có số liệu sớm, không chờ scheduler | [Đề xuất] |
| TS-5 | Coverage là sàn theo vùng rủi ro | Dồn nỗ lực vào policy/scheduler | [Đề xuất] |
| TS-6 | Test container báo `SKIPPED-NO-DOCKER`, không giả pass | Docker daemon chưa chạy là sự thật | [Đã xác minh] |

## 5. Dependencies

- Docker daemon chạy (Testcontainers, security container, sandbox).
- Claude Code CLI đã login (P-C); Codex CLI cài + login (P-D) — Owner.
- `packages/testing` (FakeProvider, factories, fixture) do `test_automation` tạo ở Phase 1.
- Runner CI có Docker; self-hosted runner nếu muốn nightly real-provider [Cần người dùng phê duyệt].
- Phê duyệt chạy T1-T4 và quota (Owner).

## 6. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Fixture golden lỗi thời khi CLI cập nhật | Parser hỏng thật nhưng test xanh | Header version CLI; job nightly P-C so khớp golden; pin version CLI trong CI |
| Quota cạn khi chạy test real/T1-T4 | Mất bằng chứng | Chạy đầu cửa sổ quota, dừng khi RATE_LIMIT |
| Mock lẫn vào production | Vi phạm yêu cầu cốt lõi | Quy tắc 3.5 (1-4) + kiểm bundle |
| Test Windows-specific không chạy được trên CI Linux | Lỗi lọt | Runner Windows + máy dev |
| 18-concurrency thất bại do RAM 16 GB | Không chứng minh được | Bước tăng dần, host guard, nhận max thấp hơn theo D-010 |
| Flaky E2E vì realtime/timing | Mất niềm tin CI | Chờ theo event/`expect.poll`, không `sleep` |
| Codex chưa cài | AC-022 và adapter Codex UNVERIFIED | Ghi rõ nhãn, không báo đạt |

## 7. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Có dùng self-hosted runner có login OAuth cho nightly real-provider không (rủi ro đưa credential lên runner; ngược D-008 tinh thần)? Phương án thay thế: chỉ chạy tay trên máy Owner.
2. [Cần người dùng phê duyệt] Ngưỡng coverage và thời gian CI (3.8, 3.10).
3. [Cần người dùng phê duyệt] Thời điểm và ngân sách quota cho T1-T4.
4. Chưa có cách đo quota còn lại của gói team từ CLI (`MULTI_AGENT_SCALABILITY.md` §6) nên không tự động hoá "dừng khi sắp cạn".
5. Cách kiểm thử Codex `mxc` sandbox trên build Windows 26300 chưa xác minh (`SECURITY.md` R10).
6. Chưa chốt có cần k6/load test cho API (NFR hiệu năng) hay chỉ benchmark nội bộ.
7. Khoảng trống truy vết: ACCEPTANCE_CRITERIA có 100 AC nhưng khoảng trống số (AC-016…020, 035…040…) là chỗ dự trữ; test plan chỉ phủ AC đã viết.

## 8. Acceptance Criteria (cho chiến lược test)

- [ ] `pnpm test`, `test:int`, `test:e2e` tồn tại và chạy xanh trong CI ở cuối Phase 1.
- [ ] Mọi AC của nhóm đang thi công có test ánh xạ và nhãn bằng chứng (bảng truy vết sinh tự động hoặc tay).
- [ ] Kiểm tra bundle production không chứa FakeProvider (test âm có trong CI).
- [ ] Backend từ chối `AIW_PROVIDER_MODE=fake` ngoài `NODE_ENV=test` (test).
- [ ] Bộ P-C chạy thật được ít nhất một lần với Claude Code và lưu log.
- [ ] Báo cáo T7 tồn tại và đạt tiêu chí 3.6 trước khi bật cấu hình concurrency > 5 mặc định.
- [ ] Gate bảo mật (coverage ≥ 90%, 100% pass) được cấu hình thành required check.
- [ ] Test container được báo `SKIPPED-NO-DOCKER` thay vì pass giả khi daemon tắt.
