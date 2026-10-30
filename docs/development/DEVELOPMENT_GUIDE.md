# DEVELOPMENT GUIDE — AI WORKSPACE

Trạng thái: DRAFT (Phase 0). Ngày: 2026-10-30. Tác giả: Delivery/Engineering Lead.
Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].
Liên quan: `ENVIRONMENT_SETUP.md`, `CI_CD.md`, `DEPLOYMENT.md`, `TESTING.md`, `CODING_STANDARDS.md`, `../IMPLEMENTATION_ROADMAP.md`, `../architecture/WORKSPACE_AND_GIT_ISOLATION.md`, `../../DECISION_LOG.md`.

---

## 1. Mục đích

Quy định cách một developer hoặc một AI agent làm việc trong repo `ai-workspace-app` từ Phase 1 trở đi: cấu trúc thư mục, quy trình làm việc, quy ước nhánh, cách nhiều agent AI cùng sửa repo mà không giẫm chân nhau, thứ tự đọc tài liệu cho agent mới, và định nghĩa "Done". Tài liệu này là hợp đồng vận hành giữa Orchestrator, các agent và Owner.

## 2. Phạm vi

- Trong phạm vi: repo của chính nền tảng AI WORKSPACE (monorepo). Quy trình dev, review, merge, bằng chứng hoàn thành.
- Ngoài phạm vi: cấu hình máy dev (xem `ENVIRONMENT_SETUP.md`), pipeline CI (xem `CI_CD.md`), quy trình của project do nền tảng sinh ra cho người dùng (xem `WORKSPACE_AND_GIT_ISOLATION.md` §8), chuẩn mã (xem `CODING_STANDARDS.md`), chiến lược test (xem `TESTING.md`).
- Phase 0 chỉ có tài liệu. Mọi mô tả cấu trúc mã dưới đây là **thiết kế mục tiêu** cho Phase 1, chưa có mã nào tồn tại. [Đã xác minh: thư mục hiện chỉ chứa `docs/` và các file quản trị]

## 3. Nội dung chi tiết

### 3.1 Cấu trúc repo mục tiêu [Đề xuất]

Nguồn: `CI_CD.md` §3 (apps/api, apps/web, apps/desktop, packages/*, infra/*) và `DECISION_LOG.md` D-001 (pnpm workspaces, NestJS, React 19 + Vite).

```
ai-workspace-app/
├── AGENTS.md                      # điểm vào cho agent AI (Phase 1, P1-03) — chưa tồn tại
├── CLAUDE.md                      # trỏ về AGENTS.md (nếu dùng Claude Code)
├── DECISION_LOG.md  WORK_PLAN.md  PROJECT_STATUS.md  IMPLEMENTATION_LOG.md  KNOWN_ISSUES.md
├── package.json  pnpm-workspace.yaml  .nvmrc  tsconfig.base.json
├── apps/
│   ├── api/                       # NestJS backend (modular monolith)
│   │   └── src/modules/           # agents, organization, providers, skills, workflows,
│   │                              # scheduler, tasks, approvals, workspace, sandbox,
│   │                              # security, events, usage, audit, ...
│   ├── web/                       # React 19 + Vite + Tailwind v4 + shadcn/ui
│   │   └── src/{app,routes,features,components/{ui,app},office,lib,stores}
│   └── desktop/                   # Tauri 2 shell (Phase 7, sidecar = apps/api)
├── packages/
│   ├── contracts/                 # OpenAPI + Zod/DTO schema dùng chung FE/BE
│   ├── events/                    # event catalogue + schema (FR-116/117), versioned
│   ├── db/                        # Drizzle schema, migration, seed (18 AgentTemplate)
│   ├── provider-sdk/              # interface ProviderAdapter + 13 operation chuẩn (FR-081)
│   ├── adapters-claude/  adapters-codex/  adapters-openhands/
│   ├── policy/                    # command policy, normalizer, manifest check, redaction
│   ├── ui/                        # (tuỳ chọn) primitives dùng chung
│   └── testing/                   # fixtures, fake provider stream, harness T1-T7
├── infra/
│   ├── compose/                   # Postgres 16 + Redis 7 (loopback only)
│   └── images/                    # sandbox images (Dockerfile + trivy scan)
├── seed/                          # agent templates, skills, workflow templates (DATA, không phải code)
├── bench/                         # script benchmark concurrency (ngoài CI thường)
├── .github/workflows/             # ci.yml, images.yml, release.yml, codeql.yml
└── docs/                          # tài liệu (giữ nguyên cấu trúc hiện tại)
```

Quy tắc ranh giới (enforce bằng ESLint `no-restricted-imports` + `dependency-cruiser`):

| Từ | Được import | Cấm |
|---|---|---|
| `apps/web` | `packages/contracts`, `packages/events`, `packages/ui` | `apps/api`, `packages/db`, `packages/policy`, `adapters-*` |
| `apps/api/modules/X` | `packages/*`, module khác qua service/public API của module đó | import trực tiếp repository/entity của module khác |
| `packages/adapters-*` | `packages/provider-sdk`, `packages/events` | `apps/*`, `packages/db` |
| `packages/policy` | không phụ thuộc framework | NestJS, Drizzle |

### 3.2 Quy trình làm việc cho một task [Đề xuất]

1. **Nhận task** từ Task Board nội bộ (`IMPLEMENTATION_ROADMAP.md`, ID dạng `P1-07`). Task phải khai báo: `owns` (file/glob được ghi), `reads` (chỉ đọc), `depends_on`, `AC` tham chiếu.
2. **Tạo worktree + branch** (mục 3.3). Không làm việc trực tiếp trên `main`.
3. **Đọc trước khi viết** (mục 3.5). Kiểm tra `DECISION_LOG.md` xem quyết định liên quan đã duyệt chưa; nếu "Chờ duyệt" thì làm theo hướng đề xuất nhưng ghi rõ giả định trong PR.
4. **Viết test trước hoặc cùng lúc** (TDD cho logic domain; xem `TESTING.md`).
5. **Chạy local gate**: `pnpm lint && pnpm typecheck && pnpm test` (rút gọn theo filter của package).
6. **Mở PR** nhỏ (≤ 400 dòng thay đổi không tính lockfile/generated) với template ở mục 3.7.
7. **Review** bởi `reviewer` (agent) và, với thay đổi chạm vùng nhạy cảm (mục 3.6), thêm `security`.
8. **Orchestrator kiểm diff + chạy test độc lập** (WORK_PLAN §3: "Orchestrator kiểm diff + chạy test trước khi tích hợp") rồi squash-merge.
9. **Cập nhật** `IMPLEMENTATION_LOG.md` (một dòng: task, commit, bằng chứng) và `PROJECT_STATUS.md` khi đóng một nhóm task.

### 3.3 Branching và commit [Đề xuất]

| Loại | Mẫu tên | Ghi chú |
|---|---|---|
| Nhánh mặc định | `main` | luôn xanh, bảo vệ bằng ruleset (`CI_CD.md` §5) |
| Tích hợp phase | `integration/phase-<n>` | chỉ dùng khi một phase có ≥ 4 nhóm song song cần ghép trước khi vào `main` |
| Task do agent | `agent/<role>/<task-id>` ví dụ `agent/backend/P1-51` | khớp quy ước `agent/<slug>/<task-id>` của D-009 |
| Task do người | `feat/<scope>-<slug>`, `fix/<scope>-<slug>`, `docs/<slug>` | |
| Hotfix | `hotfix/<slug>` | chỉ khi `main` hỏng |

- Commit theo Conventional Commits (chi tiết `CODING_STANDARDS.md` §3.9), có trailer `Task-Id` và `Agent`.
- Merge: **squash merge**, history tuyến tính (`CI_CD.md` §5).
- Không `git push --force` lên nhánh dùng chung; agent chỉ force-push nhánh `agent/*` của chính mình khi rebase.
- Không agent nào tự merge vào `main` khi chưa có PR được duyệt và `ci-ok` xanh. [Đề xuất, đồng bộ D-009: "không tự merge vào main"]
- Về attribution: quy ước dòng `Co-Authored-By` cho commit do AI tạo **tuân theo chính sách của Owner** (xem 3.4.7 và mục Vấn đề chưa giải quyết 1).

### 3.4 Nhiều AI agent làm việc trong cùng repo [Đề xuất]

Nguyên tắc đồng bộ với `WORK_PLAN.md` §3 và `WORKSPACE_AND_GIT_ISOLATION.md`: mỗi task một worktree một branch, không hai agent ghi cùng file.

3.4.1 **Vai trò (18 role seed, `AGENT_ORGANIZATION.md` §4):** `pm, ba, architect, uiux, frontend, backend, db_infra, qa, security, reviewer, research, integration, performance, devsecops, docs, ux_research, test_automation, release`. Trong giai đoạn xây nền tảng, các role này được dùng như **nhãn trách nhiệm** gắn cho task trong roadmap; agent thực tế là các Claude Code subagent do Orchestrator khởi chạy (giới hạn 10 đồng thời, WORK_PLAN đầu trang) [Nhất quán với WORK_PLAN.md]. Codex chưa cài nên chưa agent nào dùng Codex. [Đã xác minh]

3.4.2 **File ownership theo glob.** Mỗi task có manifest đơn giản trong mô tả task:

```yaml
task: P1-51
role: backend
owns:   ["apps/api/src/modules/agents/**", "apps/api/test/agents/**"]
reads:  ["packages/contracts/**", "packages/events/**", "docs/architecture/AGENT_ORGANIZATION.md"]
forbidden: ["packages/db/**", ".github/**", "pnpm-lock.yaml"]
```

Bảng ownership mặc định theo role (rút gọn, chi tiết `AGENT_ORGANIZATION.md` §4.1):

| Role | Sở hữu mặc định | Chỉ đọc |
|---|---|---|
| backend | `apps/api/src/modules/**` (module được giao) | contracts, events |
| frontend | `apps/web/src/**` (feature được giao) | contracts, ui |
| db_infra | `packages/db/**`, `infra/compose/**`, `seed/**` | modules |
| integration | `packages/adapters-*/**`, `packages/provider-sdk/**` | events |
| uiux | `docs/design/**`, token CSS | web |
| test_automation | `apps/web/e2e/**`, `packages/testing/**` | tất cả |
| devsecops | `.github/**`, `infra/images/**` | tất cả |
| security | `packages/policy/**` (đồng sở hữu với backend theo quy tắc 3.6) | tất cả |
| architect | `packages/contracts/**`, `packages/events/**`, `docs/adr/**` | tất cả |
| docs | `docs/**` trừ vùng của uiux/architect | tất cả |

3.4.3 **File dùng chung (hot files)** — chỉ một nơi sở hữu, người khác gửi yêu cầu qua PR vào chủ sở hữu hoặc qua Orchestrator:

| File | Chủ sở hữu | Vì sao |
|---|---|---|
| `pnpm-lock.yaml`, `package.json` gốc | Orchestrator (hoặc `devsecops`) | xung đột merge khó giải; agent thêm dependency qua yêu cầu `deps-request` trong PR |
| `packages/contracts/**`, `packages/events/**` | `architect` | hợp đồng FE/BE; thay đổi đi qua RFC ngắn (3.4.5) |
| Migration `packages/db/migrations/**` | `db_infra` | thứ tự migration tuyến tính; mỗi agent khác chỉ đề xuất schema diff |
| `.github/workflows/**` | `devsecops` | |
| `tsconfig.base.json`, `eslint.config.*` | Orchestrator | |

3.4.4 **Bắt đầu và kết thúc một phiên agent:**
- Bắt đầu: đọc theo thứ tự ở 3.5; chạy `git fetch && git rebase origin/main` trên branch của mình; xác nhận `owns` không giao với PR đang mở (`gh pr list --search "path:<glob>"` hoặc bảng claim trong `IMPLEMENTATION_LOG.md`).
- Kết thúc: chạy local gate; ghi bằng chứng (lệnh + kết quả rút gọn) vào PR; không để file tạm, không commit `.env`, không commit log của CLI.

3.4.5 **Thay đổi hợp đồng (contract change):** mọi thay đổi `contracts`/`events` phải (a) có mục "Contract change" trong PR, (b) giữ tương thích ngược hoặc nâng version, (c) cập nhật test contract (`TESTING.md` §3.4), (d) thông báo cho owner của consumer qua Orchestrator. Event schema có trường `schemaVersion`.

3.4.6 **Xử lý xung đột:** agent KHÔNG tự giải conflict ngoài vùng `owns`. Nếu rebase gây conflict ở file không thuộc mình, dừng, ghi vào PR và báo Orchestrator (đồng bộ `WORKSPACE_AND_GIT_ISOLATION.md` §8.2).

3.4.7 **Credential và dữ liệu nhạy cảm:** agent không đọc `~/.claude/.credentials.json` hay `auth.json` của Codex, không in biến môi trường vào log, không dán token vào PR (D-008). Hook `gitleaks` pre-commit là bắt buộc. [Đề xuất]

3.4.8 **Giới hạn song song khi phát triển:** tối đa 10 subagent đồng thời (giới hạn hiện tại của Orchestrator [Nhất quán với WORK_PLAN.md]). Chi phí và quota là hạn chế thực (`MULTI_AGENT_SCALABILITY.md` §6); do đó roadmap đánh dấu song song/tuần tự và Orchestrator không khởi chạy quá giới hạn.

3.4.9 **Phân biệt hai thứ "18 agent":** (a) 18 role trong sản phẩm là **dữ liệu seed** cho người dùng cuối (D-016); (b) agent AI đang xây repo này. Mã nguồn không được chứa danh sách 18 role hardcode; chúng nằm trong `seed/` và DB.

### 3.5 Thứ tự đọc cho agent mới [Đề xuất]

Mọi agent mới (hoặc phiên mới) đọc đúng thứ tự dưới đây, dừng khi đã đủ cho task nhưng không bỏ bước 1–4.

| # | Tài liệu | Mục đích |
|---|---|---|
| 1 | `AGENTS.md` (gốc repo; tạo ở P1-03, **chưa tồn tại** ở Phase 0) | luật vận hành ngắn, link tới các file dưới |
| 2 | `DECISION_LOG.md` | quyết định D-001…D-016 và trạng thái duyệt |
| 3 | `WORK_PLAN.md`, `PROJECT_STATUS.md`, `KNOWN_ISSUES.md` | đang ở phase nào, việc dang dở |
| 4 | `docs/development/DEVELOPMENT_GUIDE.md` (file này), `CODING_STANDARDS.md`, `TESTING.md` | cách làm |
| 5 | `docs/IMPLEMENTATION_ROADMAP.md` | task, AC, phụ thuộc |
| 6 | Docs miền của task: `architecture/*`, `DATABASE_DESIGN.md`, `SECURITY.md` | thiết kế |
| 7 | `docs/product/FUNCTIONAL_REQUIREMENTS.md`, `ACCEPTANCE_CRITERIA.md` | FR/AC cần thoả |
| 8 | Với FE: `design/DESIGN_SYSTEM.md`, `DESIGN_TOKENS.md`, `COMPONENT_SPECIFICATION.md`, `SCREEN_SPECIFICATION.md` | UI |
| 9 | Với provider: `research/CLAUDE_CODE_INTEGRATION.md`, `CODEX_INTEGRATION.md` | tích hợp |
| 10 | `docs/PROJECT_RISKS.md` | rủi ro liên quan task |

Nội dung tối thiểu của `AGENTS.md` (để P1-03 viết) [Đề xuất]: (1) lệnh build/test/lint; (2) đường dẫn đọc ở bảng trên; (3) quy tắc ownership và hot files; (4) danh sách pattern bị cấm (mục 3.8 `CODING_STANDARDS.md`); (5) định nghĩa Done (3.9 dưới đây); (6) cách báo cáo kết quả (≤ 150 từ + đường dẫn tuyệt đối file đã đổi).

### 3.6 Vùng nhạy cảm — yêu cầu review thêm [Đề xuất]

Các đường dẫn sau cần review của `security` và coverage ≥ 90% (đồng bộ `SECURITY.md` §9 gate):

`packages/policy/**`, `apps/api/src/modules/{security,sandbox,providers,audit}/**`, `packages/adapters-*/**`, hook script sinh cho CLI, cấu hình container, redaction middleware.

### 3.7 Mẫu PR [Đề xuất]

```
## Task
P1-51 — Agents module CRUD (FR-021, FR-022, FR-031; AC-001, AC-002, AC-014)
## Owns / Reads
## Thay đổi chính
## Contract change? (có/không, nếu có: version + consumer bị ảnh hưởng)
## Bằng chứng
- pnpm lint: <kết quả>   - pnpm typecheck: <kết quả>
- pnpm test --filter api: <x passed>   - e2e/AC: <id đã chạy>
## Giả định / quyết định chờ duyệt (D-xxx)
## Rủi ro còn lại
```

### 3.8 Cấu hình cục bộ và lệnh chuẩn [Đề xuất]

| Lệnh | Tác dụng |
|---|---|
| `pnpm install --frozen-lockfile` | cài dependency |
| `pnpm doctor` | kiểm tra môi trường (`ENVIRONMENT_SETUP.md` §8) |
| `pnpm dev` | chạy api + web (Postgres/Redis qua compose) |
| `pnpm lint` / `pnpm typecheck` | ESLint, `tsc --noEmit` (project references) |
| `pnpm test` / `pnpm test:int` / `pnpm test:e2e` | Vitest unit / integration / Playwright |
| `pnpm db:migrate` / `pnpm db:seed` | Drizzle migration và seed |
| `pnpm test:provider:real` | provider test với CLI thật, chỉ chạy tay hoặc nightly |
| `pnpm bench:concurrency -- --n 5` | benchmark T1-T7 (cần phê duyệt quota, `TESTING.md` §3.6) |

Lưu ý Windows [Đã xác minh từ `ENVIRONMENT_SETUP.md`]: bật long paths, dùng Git for Windows, shell mặc định PowerShell/Git Bash; script trong `package.json` phải chạy được trên cả hai (dùng `cross-env`, `node` script, tránh `rm -rf`, `export`).

### 3.9 Definition of Done (DoD) [Đề xuất]

Một task chỉ được đánh dấu COMPLETED khi **tất cả** thoả (WORK_PLAN §3: "Không đánh dấu COMPLETED khi chưa có bằng chứng kiểm thử"):

1. Mã đạt `lint`, `typecheck`, `test` xanh trên CI (không chỉ local).
2. Test mới cho hành vi mới; test cho AC được tham chiếu đã chạy và pass (ghi ID AC trong PR).
3. Không vi phạm pattern cấm: không hardcode agent/workflow, không mock data trong luồng sản phẩm, không credential ở frontend (`CODING_STANDARDS.md` §3.8).
4. Không secret mới (gitleaks sạch); không `console.log` còn sót; log đã mask.
5. DTO có validation; lỗi trả theo mã lỗi chuẩn.
6. Event liên quan đã phát và được test (nếu task đổi trạng thái domain, FR-025/FR-118).
7. Tài liệu cập nhật nếu đổi hành vi/contract (docs, OpenAPI, ADR).
8. Với UI: đủ các state loading/empty/error, axe không lỗi nghiêm trọng, phím tắt/focus theo `DESIGN_SYSTEM.md` §3.0.
9. Với tính năng provider: có bằng chứng chạy với CLI thật (log, session id) hoặc ghi rõ "chưa kiểm chứng thật" và **không** đánh dấu COMPLETED mà đánh dấu `IMPLEMENTED-UNVERIFIED`. [Đề xuất]
10. Đã được review (và `security` nếu thuộc 3.6); Orchestrator đã chạy lại test độc lập.
11. `IMPLEMENTATION_LOG.md` có dòng bằng chứng (commit, lệnh, kết quả).

Trạng thái task: `TODO → IN_PROGRESS → IN_REVIEW → COMPLETED`; thêm `IMPLEMENTED-UNVERIFIED` và `BLOCKED(<lý do>)`.

### 3.10 Báo cáo hoàn thành của agent [Đề xuất]

Báo cáo ≤ 150 từ: (1) file đã tạo/sửa (đường dẫn tuyệt đối), (2) lệnh test đã chạy + kết quả, (3) AC đã thoả, (4) khoảng trống/rủi ro. Không tạo file báo cáo `.md` riêng ngoài `IMPLEMENTATION_LOG.md`.

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| DG-1 | Monorepo pnpm workspaces (không Turborepo), ranh giới import enforce bằng lint | Chặn FE gọi trực tiếp BE/DB, giữ contract rõ | D-001 APPROVED (Owner, 2026-10-30) |
| DG-2 | Mỗi task một worktree/branch `agent/<role>/<task-id>` | Cùng cơ chế sản phẩm sẽ dùng (D-009), tự "ăn thức ăn của mình" | [Đề xuất] |
| DG-3 | Ownership theo glob + hot files một chủ | Tránh xung đột lockfile/migration/contract | [Đề xuất] |
| DG-4 | Squash merge, history tuyến tính | Dễ revert theo task, khớp `CI_CD.md` §5 | [Đề xuất] |
| DG-5 | DoD yêu cầu bằng chứng và trạng thái `IMPLEMENTED-UNVERIFIED` | Tránh báo hoàn thành sai khi Codex chưa cài/Docker chưa chạy | [Đề xuất] |
| DG-6 | `AGENTS.md` là điểm vào duy nhất cho agent | Giảm lệch chỉ dẫn giữa các agent | [Đề xuất] |
| DG-7 | Giới hạn 10 subagent đồng thời khi phát triển | Giới hạn hiện tại của Orchestrator + quota chung | [Nhất quán với WORK_PLAN.md] |

## 5. Dependencies

- `DECISION_LOG.md` D-001, D-009, D-016 được duyệt (hiện "Chờ duyệt").
- `CI_CD.md` (required checks), `ENVIRONMENT_SETUP.md` (`pnpm doctor`), `WORKSPACE_AND_GIT_ISOLATION.md` (quy ước nhánh/commit).
- GitHub repo `<org>/ai-workspace-app` đã tạo [Giả định: chưa tồn tại; thư mục hiện không phải git repo].
- `AGENTS.md` do P1-03 tạo.

## 6. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Hai agent sửa cùng file do manifest không đủ chi tiết | Conflict, mất công | Bảng hot files, kiểm tra `gh pr list` trước khi bắt đầu |
| Agent báo COMPLETED không có bằng chứng | Nợ chất lượng ẩn | DoD 11 điểm, Orchestrator chạy lại test |
| Quota subscription cạn khi nhiều agent chạy | Đứng việc | Giới hạn 10 agent, ưu tiên task theo đường găng |
| Thư mục chưa là git repo | Không dùng được worktree | P1-01 khởi tạo repo trước mọi việc |
| Lệnh shell không portable Windows | CI xanh nhưng dev Windows hỏng | Chạy CI cả `windows-latest` cho job `unit` [Đề xuất] |

## 7. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Chính sách attribution: ghi `Co-Authored-By` cho commit do AI tạo hay không? Quy ước sản phẩm (`WORKSPACE_AND_GIT_ISOLATION.md` §9) có trailer `Co-Authored-By: Claude`; cần Owner xác nhận cho repo nền tảng.
2. [Cần người dùng phê duyệt] GitHub remote: org/tên repo, có bật ruleset và `CODEOWNERS` không.
3. [Giả định] Solo developer + AI agents: số reviewer người = 1 (Owner). Cần xác nhận quy tắc self-approve (`CI_CD.md` §5).
4. Chưa có `AGENTS.md` và `CLAUDE.md`; nội dung 3.5 là đề xuất.
5. Mâu thuẫn số liệu chưa giải quyết ở tài liệu khác ảnh hưởng dev: `task_status` HOA vs thường (`DECISION_LOG.md` mục 5); 12 trạng thái agent (FR-024) vs danh sách trong `SCREEN_SPECIFICATION.md` §2. Cần Architect chốt trước P1-04 (contracts).
6. ORM: Drizzle (stable 0.45.x) theo D-002 (APPROVED, Owner 2026-10-30). Tác vụ bắt buộc trước khi khóa schema (chưa thực hiện): spike partition `agent_events` và FK vòng bằng SQL tay.

## 8. Acceptance Criteria (cho tài liệu và quy trình)

- [ ] Agent mới chỉ dựa vào mục 3.5 có thể tìm đúng tài liệu cho một task Phase 1 mà không cần hỏi thêm.
- [ ] `AGENTS.md` (P1-03) bao phủ đủ 6 mục ở 3.5 và được Owner duyệt.
- [ ] Ruleset `main` bật đúng theo 3.3 và `CI_CD.md` §5 (kiểm bằng `gh api`).
- [ ] Một PR thử nghiệm từ agent đi hết quy trình 3.2 với DoD 3.9 điền đủ bằng chứng (dry-run ở P1-72 của roadmap).
- [ ] Lint ranh giới import (3.1) fail khi `apps/web` import `apps/api` (test âm).
- [ ] Không file nào trong `src/` chứa danh sách 18 role hardcode (kiểm bằng rule ở `CODING_STANDARDS.md`).
